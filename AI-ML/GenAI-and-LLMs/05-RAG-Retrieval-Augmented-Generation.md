# Chapter 05: RAG — Retrieval-Augmented Generation

## Table of Contents
- [What is RAG?](#what-is-rag)
- [Why RAG Matters](#why-rag-matters)
- [How RAG Works — Complete Pipeline](#how-rag-works--complete-pipeline)
- [Embeddings Deep Dive](#embeddings-deep-dive)
- [Vector Databases](#vector-databases)
- [Chunking Strategies](#chunking-strategies)
- [RAG Pipeline Implementation](#rag-pipeline-implementation)
- [Advanced RAG Techniques](#advanced-rag-techniques)
- [Evaluation and Metrics](#evaluation-and-metrics)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is RAG?

### Simple Explanation (ELI15)

Imagine you're taking an open-book exam. Instead of memorizing everything (like a fine-tuned model), you're allowed to look up information in your textbooks during the test. RAG gives an LLM an "open book" — before answering a question, it searches through your documents to find relevant information, then crafts an answer using both its own knowledge AND the retrieved facts.

### Formal Definition

**Retrieval-Augmented Generation (RAG)** is a technique that enhances LLM responses by retrieving relevant information from an external knowledge base and injecting it into the prompt context before generation. This grounds the model's output in factual, up-to-date information without requiring retraining.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     RAG CONCEPTUAL FLOW                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  User Query: "What is our company's refund policy?"                 │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────┐     ┌──────────────────────────┐              │
│  │ Embed Query      │────►│ Search Vector Database    │              │
│  │ (Dense Vector)   │     │ (Find similar documents) │              │
│  └─────────────────┘     └──────────┬───────────────┘              │
│                                      │                              │
│                                      ▼                              │
│                           ┌──────────────────────┐                  │
│                           │ Retrieved Context:    │                  │
│                           │ "Refunds within 30   │                  │
│                           │  days, original      │                  │
│                           │  receipt required..." │                  │
│                           └──────────┬───────────┘                  │
│                                      │                              │
│                                      ▼                              │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ Augmented Prompt:                                     │          │
│  │ "Based on the following context, answer the question" │          │
│  │ Context: [retrieved docs]                             │          │
│  │ Question: "What is our refund policy?"                │          │
│  └──────────────────────┬───────────────────────────────┘          │
│                          │                                          │
│                          ▼                                          │
│                    ┌────────────┐                                   │
│                    │    LLM     │                                   │
│                    │ (Generate) │                                   │
│                    └─────┬──────┘                                   │
│                          │                                          │
│                          ▼                                          │
│  Answer: "Our refund policy allows returns within 30 days..."      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Why RAG Matters

### The Problem RAG Solves

| LLM Limitation | How RAG Fixes It |
|---------------|-----------------|
| **Knowledge Cutoff** — Model doesn't know recent events | Retrieves up-to-date documents |
| **Hallucinations** — Makes up plausible-sounding facts | Grounds answers in real sources |
| **No Private Data** — Can't access your internal docs | Indexes your proprietary knowledge |
| **No Citations** — Can't tell you WHERE it got info | Can reference exact source documents |
| **Token Limits** — Can't fit all docs in context | Retrieves only relevant chunks |

### RAG vs Fine-Tuning: When to Use What

```
┌────────────────────────────────────────────────────────────────────┐
│              RAG vs FINE-TUNING DECISION MATRIX                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  USE RAG when you need:              USE FINE-TUNING when:         │
│  • Access to changing information    • Consistent style/format     │
│  • Citation of sources               • Behavior modification       │
│  • Domain knowledge injection        • New capabilities            │
│  • Up-to-date answers                • Reduced latency             │
│  • Quick setup (no training)         • Specific reasoning patterns │
│                                                                    │
│  USE BOTH (RAG + Fine-Tuning) when:                               │
│  • Need custom behavior AND private knowledge                      │
│  • Building production chatbots with brand voice + knowledge       │
│  • Maximum accuracy on domain-specific tasks                       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Real-World Use Cases

1. **Enterprise Knowledge Base** — Chat with internal documentation, policies, SOPs
2. **Customer Support** — Answer questions using product docs and past tickets
3. **Legal Research** — Search case law and regulations to answer legal questions
4. **Medical Assistance** — Retrieve clinical guidelines for evidence-based answers
5. **Code Documentation** — Query your codebase and API docs
6. **Financial Analysis** — RAG over earnings reports, SEC filings, market data

---

## How RAG Works — Complete Pipeline

### Two Phases

```
┌─────────────────────────────────────────────────────────────────┐
│                     RAG SYSTEM ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ═══ PHASE 1: INDEXING (Offline, one-time) ═══                  │
│                                                                  │
│  Documents ──► Chunking ──► Embedding ──► Vector DB             │
│  (PDFs, docs,   (Split     (Convert to    (Store for            │
│   HTML, etc.)    into       vectors)       fast search)          │
│                  pieces)                                          │
│                                                                  │
│  ═══ PHASE 2: RETRIEVAL + GENERATION (Online, per query) ═══   │
│                                                                  │
│  Query ──► Embed ──► Search ──► Retrieve ──► Augment ──► LLM   │
│  (User      (Same    (Find      (Top-K      (Add to     (Answer │
│   question)  model)   similar)   chunks)     prompt)     query)  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Embeddings Deep Dive

### What Are Embeddings?

Embeddings convert text into dense numerical vectors that capture semantic meaning. Similar texts produce vectors that are close together in vector space.

```
┌─────────────────────────────────────────────────────────────────┐
│                EMBEDDING SPACE (simplified 2D)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "dog"  •─────── close ───────• "puppy"                         │
│                                                                  │
│  "cat"  •─── close ───• "kitten"                                │
│                                                                  │
│                                                                  │
│  "quantum physics" •                                             │
│                                                                  │
│                                                                  │
│  "banana" •                   • "apple"                          │
│                                                                  │
│  Key Insight: Semantically similar texts have                    │
│  similar vectors (small distance / high cosine similarity)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Similarity Metrics

**Cosine Similarity** (most common for text embeddings):

$$\text{cos\_sim}(\vec{a}, \vec{b}) = \frac{\vec{a} \cdot \vec{b}}{|\vec{a}| \times |\vec{b}|}$$

- Range: [-1, 1] (1 = identical, 0 = unrelated, -1 = opposite)
- Invariant to vector magnitude (only cares about direction)

**Euclidean Distance**:

$$d(\vec{a}, \vec{b}) = \sqrt{\sum_{i=1}^{n}(a_i - b_i)^2}$$

**Dot Product**:

$$\vec{a} \cdot \vec{b} = \sum_{i=1}^{n} a_i \times b_i$$

### Popular Embedding Models

| Model | Dimensions | Max Tokens | Quality | Speed | Use Case |
|-------|-----------|------------|---------|-------|----------|
| `text-embedding-3-large` (OpenAI) | 3072 | 8191 | Excellent | Fast (API) | Production |
| `text-embedding-3-small` (OpenAI) | 1536 | 8191 | Good | Fast (API) | Cost-sensitive |
| `all-MiniLM-L6-v2` (Sentence-BERT) | 384 | 256 | Good | Very Fast | On-premise/free |
| `bge-large-en-v1.5` (BAAI) | 1024 | 512 | Excellent | Medium | On-premise/free |
| `e5-large-v2` (Microsoft) | 1024 | 512 | Excellent | Medium | On-premise |
| `voyage-large-2` (Voyage AI) | 1536 | 16000 | Excellent | Fast (API) | Long docs |
| `nomic-embed-text-v1.5` | 768 | 8192 | Very Good | Fast | Open-source |

### Code: Generating Embeddings

```python
# ═══════════════════════════════════════════════════════
# Method 1: OpenAI Embeddings (API-based, easiest)
# ═══════════════════════════════════════════════════════

from openai import OpenAI
import numpy as np

client = OpenAI()  # Uses OPENAI_API_KEY env var

def get_openai_embedding(text: str, model="text-embedding-3-small") -> list:
    """Get embedding from OpenAI API."""
    response = client.embeddings.create(
        input=text,
        model=model
    )
    return response.data[0].embedding  # Returns list of floats

# Example
embedding = get_openai_embedding("What is machine learning?")
print(f"Dimensions: {len(embedding)}")  # 1536

# Batch embeddings (more efficient)
def get_batch_embeddings(texts: list, model="text-embedding-3-small") -> list:
    """Get embeddings for multiple texts in one API call."""
    response = client.embeddings.create(
        input=texts,
        model=model
    )
    return [item.embedding for item in response.data]


# ═══════════════════════════════════════════════════════
# Method 2: Sentence Transformers (Local, Free)
# ═══════════════════════════════════════════════════════

from sentence_transformers import SentenceTransformer

# Load model (downloads once, runs locally)
model = SentenceTransformer("all-MiniLM-L6-v2")

# Single text
embedding = model.encode("What is machine learning?")
print(f"Shape: {embedding.shape}")  # (384,)

# Batch of texts (automatically batched for GPU)
texts = [
    "Machine learning is a subset of AI",
    "Deep learning uses neural networks",
    "The weather is sunny today"
]
embeddings = model.encode(texts, show_progress_bar=True)
print(f"Shape: {embeddings.shape}")  # (3, 384)

# Compute similarity
from sentence_transformers.util import cos_sim
similarity = cos_sim(embeddings[0], embeddings[1])
print(f"ML vs DL similarity: {similarity.item():.4f}")  # ~0.75
similarity = cos_sim(embeddings[0], embeddings[2])
print(f"ML vs Weather similarity: {similarity.item():.4f}")  # ~0.10


# ═══════════════════════════════════════════════════════
# Method 3: Hugging Face Transformers (Most Control)
# ═══════════════════════════════════════════════════════

from transformers import AutoTokenizer, AutoModel
import torch
import torch.nn.functional as F

def get_hf_embedding(texts: list, model_name="BAAI/bge-large-en-v1.5"):
    """Generate embeddings using any HuggingFace model."""
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModel.from_pretrained(model_name)
    
    # Tokenize
    encoded = tokenizer(
        texts, padding=True, truncation=True,
        max_length=512, return_tensors="pt"
    )
    
    # Forward pass (no gradient needed for inference)
    with torch.no_grad():
        outputs = model(**encoded)
    
    # Use CLS token or mean pooling
    # Mean pooling (generally better)
    attention_mask = encoded["attention_mask"]
    token_embeddings = outputs.last_hidden_state
    input_mask_expanded = attention_mask.unsqueeze(-1).expand(token_embeddings.size()).float()
    embeddings = torch.sum(token_embeddings * input_mask_expanded, 1) / torch.clamp(
        input_mask_expanded.sum(1), min=1e-9
    )
    
    # Normalize
    embeddings = F.normalize(embeddings, p=2, dim=1)
    
    return embeddings.numpy()
```

---

## Vector Databases

### What is a Vector Database?

A vector database is a specialized database optimized for storing, indexing, and querying high-dimensional vectors. It uses approximate nearest neighbor (ANN) algorithms to find similar vectors in milliseconds, even with millions of entries.

### Vector Database Comparison

| Database | Type | Scalability | Cost | Best For |
|----------|------|-------------|------|----------|
| **ChromaDB** | Embedded | Small-Medium | Free | Prototyping, small apps |
| **FAISS** | Library | Large | Free | Research, on-premise |
| **Pinecone** | Managed Cloud | Very Large | Paid | Production, zero-ops |
| **Weaviate** | Self-hosted/Cloud | Large | Free/Paid | Hybrid search |
| **Qdrant** | Self-hosted/Cloud | Large | Free/Paid | Performance-critical |
| **Milvus** | Self-hosted | Very Large | Free | Enterprise scale |
| **pgvector** | PostgreSQL ext. | Medium | Free | Existing Postgres infra |

### How Vector Search Works (ANN Algorithms)

```
┌─────────────────────────────────────────────────────────────────┐
│              APPROXIMATE NEAREST NEIGHBOR (ANN)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Brute Force (Exact):  O(n) — check every vector                │
│  → Works for <10K vectors, too slow for millions                 │
│                                                                  │
│  HNSW (Hierarchical Navigable Small World):                      │
│  → Builds a graph of connected vectors at multiple layers        │
│  → O(log n) search — great for most use cases                   │
│  → Used by: Qdrant, Weaviate, pgvector                          │
│                                                                  │
│  IVF (Inverted File Index):                                      │
│  → Clusters vectors, only searches relevant clusters             │
│  → Fast but less accurate                                        │
│  → Used by: FAISS, Milvus                                       │
│                                                                  │
│  Tradeoff: Speed ◄──────────────────────► Accuracy              │
│            IVF-PQ    IVF-Flat    HNSW     Brute Force           │
│            Fastest   Fast        Good     Slowest                │
│            Least     Good        Best     Perfect                │
│            Accurate  Accuracy    Accuracy Accuracy               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Code: Working with Vector Databases

```python
# ═══════════════════════════════════════════════════════
# ChromaDB — Best for prototyping (embedded, zero-config)
# ═══════════════════════════════════════════════════════

import chromadb
from chromadb.utils import embedding_functions

# Create persistent client (data survives restarts)
client = chromadb.PersistentClient(path="./chroma_db")

# Use sentence transformer for embeddings (free, local)
embedding_fn = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2"
)

# Create or get collection
collection = client.get_or_create_collection(
    name="knowledge_base",
    embedding_function=embedding_fn,
    metadata={"hnsw:space": "cosine"}  # Use cosine similarity
)

# Add documents
documents = [
    "Machine learning is a subset of artificial intelligence.",
    "Neural networks are inspired by biological brains.",
    "Python is the most popular language for data science.",
    "Transfer learning reuses pre-trained model knowledge.",
    "GPT stands for Generative Pre-trained Transformer."
]

collection.add(
    documents=documents,
    ids=[f"doc_{i}" for i in range(len(documents))],
    metadatas=[{"source": "textbook", "chapter": i} for i in range(len(documents))]
)

# Query
results = collection.query(
    query_texts=["How does deep learning work?"],
    n_results=3,  # Top 3 most relevant
    include=["documents", "distances", "metadatas"]
)

print("Most relevant documents:")
for doc, dist in zip(results["documents"][0], results["distances"][0]):
    print(f"  [{dist:.4f}] {doc}")


# ═══════════════════════════════════════════════════════
# FAISS — Best for large-scale, on-premise
# ═══════════════════════════════════════════════════════

import faiss
import numpy as np
from sentence_transformers import SentenceTransformer

# Generate embeddings
model = SentenceTransformer("all-MiniLM-L6-v2")
documents = ["doc1...", "doc2...", "doc3..."]  # Your documents
embeddings = model.encode(documents).astype("float32")

# Create FAISS index
dimension = embeddings.shape[1]  # 384 for MiniLM

# Option 1: Flat index (exact search, good for < 100K vectors)
index = faiss.IndexFlatIP(dimension)  # Inner Product (use with normalized vectors)
faiss.normalize_L2(embeddings)        # Normalize for cosine similarity
index.add(embeddings)

# Option 2: IVF index (approximate, good for > 100K vectors)
nlist = 100  # Number of clusters
quantizer = faiss.IndexFlatIP(dimension)
index = faiss.IndexIVFFlat(quantizer, dimension, nlist, faiss.METRIC_INNER_PRODUCT)
index.train(embeddings)  # Must train before adding
index.add(embeddings)
index.nprobe = 10  # Search 10 nearest clusters

# Search
query = "How does deep learning work?"
query_embedding = model.encode([query]).astype("float32")
faiss.normalize_L2(query_embedding)

distances, indices = index.search(query_embedding, k=5)  # Top 5
print(f"Top results: {indices[0]}")
print(f"Scores: {distances[0]}")

# Save/Load index
faiss.write_index(index, "my_index.faiss")
index = faiss.read_index("my_index.faiss")


# ═══════════════════════════════════════════════════════
# Pinecone — Best for production (managed, scalable)
# ═══════════════════════════════════════════════════════

from pinecone import Pinecone, ServerlessSpec

# Initialize
pc = Pinecone(api_key="your-api-key")

# Create index
pc.create_index(
    name="knowledge-base",
    dimension=1536,  # Match your embedding model
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1")
)

# Connect to index
index = pc.Index("knowledge-base")

# Upsert vectors
vectors = [
    {
        "id": "doc_1",
        "values": embedding_1,  # Your embedding list
        "metadata": {"source": "manual.pdf", "page": 5, "text": "..."}
    },
    # ... more vectors
]
index.upsert(vectors=vectors, namespace="product-docs")

# Query
results = index.query(
    vector=query_embedding,
    top_k=5,
    namespace="product-docs",
    include_metadata=True,
    filter={"source": {"$eq": "manual.pdf"}}  # Metadata filtering
)

for match in results.matches:
    print(f"Score: {match.score:.4f} | {match.metadata['text'][:100]}")
```

---

## Chunking Strategies

### Why Chunking Matters

Documents are often too long to embed as a single vector. Chunking splits them into smaller, meaningful pieces. **The quality of your chunking directly determines the quality of your retrieval.**

### Chunking Methods Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                   CHUNKING STRATEGIES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. FIXED SIZE (Naive)                                           │
│  ┌──────────┐┌──────────┐┌──────────┐┌──────────┐              │
│  │ 500 chars ││ 500 chars ││ 500 chars ││ 500 chars │              │
│  └──────────┘└──────────┘└──────────┘└──────────┘              │
│  Pro: Simple. Con: Splits sentences mid-thought                  │
│                                                                  │
│  2. FIXED SIZE WITH OVERLAP                                      │
│  ┌──────────────┐                                                │
│  │   Chunk 1     │                                               │
│  └───────┬───────┘                                               │
│          ┌┴──────────────┐                                       │
│          │   Chunk 2     │                                       │
│          └───────┬───────┘                                       │
│                  ┌┴──────────────┐                                │
│                  │   Chunk 3     │                                │
│                  └───────────────┘                                │
│  Pro: Context preserved at boundaries. Con: Redundancy           │
│                                                                  │
│  3. SEMANTIC CHUNKING                                            │
│  ┌──────────────────┐┌─────────────┐┌───────────────────────┐   │
│  │ Paragraph about A ││ Topic B     ││ Longer section on C    │   │
│  └──────────────────┘└─────────────┘└───────────────────────┘   │
│  Pro: Meaningful units. Con: Variable sizes, more complex        │
│                                                                  │
│  4. RECURSIVE (Best General Purpose)                             │
│  Split by: \n\n → \n → . → " " → character                     │
│  Pro: Respects structure. Con: Requires tuning separators        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Optimal Chunk Sizes

| Content Type | Recommended Chunk Size | Overlap |
|-------------|----------------------|---------|
| Technical documentation | 500-1000 tokens | 50-100 tokens |
| Legal documents | 300-500 tokens | 50-100 tokens |
| Conversational text | 200-400 tokens | 20-50 tokens |
| Code files | By function/class | Minimal |
| Academic papers | By section/paragraph | 50-100 tokens |

> **Pro Tip**: Chunk size should roughly match the expected query length and answer granularity. If users ask very specific questions, use smaller chunks. If they ask broad questions, use larger chunks.

### Code: Chunking Implementation

```python
# ═══════════════════════════════════════════════════════
# Chunking Strategies Implementation
# ═══════════════════════════════════════════════════════

from typing import List
import re

# ─────────────────────────────────────────────────────
# Strategy 1: Fixed Size with Overlap (Simple but effective)
# ─────────────────────────────────────────────────────

def chunk_fixed_size(text: str, chunk_size: int = 500, overlap: int = 50) -> List[str]:
    """Split text into fixed-size chunks with overlap."""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk.strip())
        start = end - overlap  # Move back by overlap amount
    return [c for c in chunks if c]  # Remove empty chunks


# ─────────────────────────────────────────────────────
# Strategy 2: Recursive Character Splitting (LangChain-style)
# ─────────────────────────────────────────────────────

def chunk_recursive(
    text: str,
    chunk_size: int = 1000,
    chunk_overlap: int = 200,
    separators: List[str] = None
) -> List[str]:
    """
    Recursively split text using a hierarchy of separators.
    Tries to keep semantically related text together.
    """
    if separators is None:
        separators = ["\n\n", "\n", ". ", " ", ""]
    
    chunks = []
    separator = separators[0]
    
    # Find the appropriate separator
    for sep in separators:
        if sep in text:
            separator = sep
            break
    
    # Split by the separator
    splits = text.split(separator) if separator else list(text)
    
    # Merge splits into chunks of appropriate size
    current_chunk = ""
    for split in splits:
        if len(current_chunk) + len(split) + len(separator) <= chunk_size:
            current_chunk += (separator if current_chunk else "") + split
        else:
            if current_chunk:
                chunks.append(current_chunk.strip())
            # If single split is too large, recursively split with next separator
            if len(split) > chunk_size and len(separators) > 1:
                sub_chunks = chunk_recursive(
                    split, chunk_size, chunk_overlap, separators[1:]
                )
                chunks.extend(sub_chunks)
                current_chunk = ""
            else:
                current_chunk = split
    
    if current_chunk:
        chunks.append(current_chunk.strip())
    
    # Add overlap
    if chunk_overlap > 0 and len(chunks) > 1:
        overlapped = [chunks[0]]
        for i in range(1, len(chunks)):
            # Prepend end of previous chunk
            prev_end = chunks[i-1][-chunk_overlap:]
            overlapped.append(prev_end + " " + chunks[i])
        chunks = overlapped
    
    return chunks


# ─────────────────────────────────────────────────────
# Strategy 3: Semantic Chunking (Advanced)
# ─────────────────────────────────────────────────────

from sentence_transformers import SentenceTransformer
import numpy as np

def chunk_semantic(
    text: str,
    model: SentenceTransformer,
    threshold: float = 0.5,
    min_chunk_size: int = 100,
    max_chunk_size: int = 2000
) -> List[str]:
    """
    Split text based on semantic similarity between sentences.
    Combines sentences that are semantically related.
    """
    # Split into sentences
    sentences = re.split(r'(?<=[.!?])\s+', text)
    
    if len(sentences) <= 1:
        return [text]
    
    # Get embeddings for each sentence
    embeddings = model.encode(sentences)
    
    # Calculate similarity between consecutive sentences
    chunks = []
    current_chunk = [sentences[0]]
    
    for i in range(1, len(sentences)):
        # Cosine similarity between current and previous sentence
        sim = np.dot(embeddings[i], embeddings[i-1]) / (
            np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[i-1])
        )
        
        current_text = " ".join(current_chunk)
        
        # Start new chunk if similarity is low AND chunk is big enough
        if sim < threshold and len(current_text) >= min_chunk_size:
            chunks.append(current_text)
            current_chunk = [sentences[i]]
        # Also start new chunk if current is getting too large
        elif len(current_text) + len(sentences[i]) > max_chunk_size:
            chunks.append(current_text)
            current_chunk = [sentences[i]]
        else:
            current_chunk.append(sentences[i])
    
    # Don't forget the last chunk
    if current_chunk:
        chunks.append(" ".join(current_chunk))
    
    return chunks


# ─────────────────────────────────────────────────────
# Strategy 4: Document-Aware Chunking (For specific formats)
# ─────────────────────────────────────────────────────

def chunk_markdown(text: str, max_chunk_size: int = 1500) -> List[dict]:
    """
    Split markdown by headers, preserving hierarchy.
    Returns chunks with metadata about their position.
    """
    # Split by headers
    sections = re.split(r'(^#{1,6}\s+.+$)', text, flags=re.MULTILINE)
    
    chunks = []
    current_headers = {}  # Track header hierarchy
    current_content = ""
    
    for section in sections:
        if re.match(r'^#{1,6}\s+', section):
            # It's a header — save previous content
            if current_content.strip():
                chunks.append({
                    "text": current_content.strip(),
                    "headers": dict(current_headers),
                    "char_count": len(current_content.strip())
                })
                current_content = ""
            
            # Update header hierarchy
            level = len(re.match(r'^(#+)', section).group(1))
            header_text = section.lstrip('#').strip()
            current_headers[f"h{level}"] = header_text
            # Clear lower-level headers
            for l in range(level + 1, 7):
                current_headers.pop(f"h{l}", None)
            
            current_content = section + "\n"
        else:
            current_content += section
    
    # Last section
    if current_content.strip():
        chunks.append({
            "text": current_content.strip(),
            "headers": dict(current_headers),
            "char_count": len(current_content.strip())
        })
    
    return chunks


# ═══════════════════════════════════════════════════════
# Usage Example
# ═══════════════════════════════════════════════════════

# Load a document
with open("document.md", "r") as f:
    text = f.read()

# Try different strategies
print("=== Fixed Size ===")
chunks_fixed = chunk_fixed_size(text, chunk_size=500, overlap=50)
print(f"Number of chunks: {len(chunks_fixed)}")

print("\n=== Recursive ===")
chunks_recursive = chunk_recursive(text, chunk_size=1000, chunk_overlap=200)
print(f"Number of chunks: {len(chunks_recursive)}")

print("\n=== Semantic ===")
model = SentenceTransformer("all-MiniLM-L6-v2")
chunks_semantic = chunk_semantic(text, model, threshold=0.5)
print(f"Number of chunks: {len(chunks_semantic)}")
```

---

## RAG Pipeline Implementation

### Complete End-to-End RAG System

```python
# ═══════════════════════════════════════════════════════════════════
# COMPLETE RAG PIPELINE — Production-Ready Implementation
# ═══════════════════════════════════════════════════════════════════

import os
from typing import List, Dict, Optional
from dataclasses import dataclass
from pathlib import Path

from openai import OpenAI
import chromadb
from chromadb.utils import embedding_functions
from sentence_transformers import SentenceTransformer

# ─────────────────────────────────────────────────────
# Configuration
# ─────────────────────────────────────────────────────

@dataclass
class RAGConfig:
    """Configuration for the RAG pipeline."""
    embedding_model: str = "all-MiniLM-L6-v2"
    llm_model: str = "gpt-4"
    chunk_size: int = 1000
    chunk_overlap: int = 200
    top_k: int = 5
    collection_name: str = "knowledge_base"
    db_path: str = "./vector_db"
    temperature: float = 0.1


# ─────────────────────────────────────────────────────
# Document Loader
# ─────────────────────────────────────────────────────

class DocumentLoader:
    """Load documents from various sources."""
    
    @staticmethod
    def load_text(file_path: str) -> str:
        with open(file_path, "r", encoding="utf-8") as f:
            return f.read()
    
    @staticmethod
    def load_pdf(file_path: str) -> str:
        """Load PDF using PyPDF2."""
        from PyPDF2 import PdfReader
        reader = PdfReader(file_path)
        text = ""
        for page in reader.pages:
            text += page.extract_text() + "\n"
        return text
    
    @staticmethod
    def load_directory(dir_path: str, extensions: list = None) -> List[Dict]:
        """Load all documents from a directory."""
        if extensions is None:
            extensions = [".txt", ".md", ".pdf"]
        
        documents = []
        for path in Path(dir_path).rglob("*"):
            if path.suffix in extensions:
                try:
                    if path.suffix == ".pdf":
                        text = DocumentLoader.load_pdf(str(path))
                    else:
                        text = DocumentLoader.load_text(str(path))
                    
                    documents.append({
                        "text": text,
                        "source": str(path),
                        "filename": path.name
                    })
                except Exception as e:
                    print(f"Error loading {path}: {e}")
        
        return documents


# ─────────────────────────────────────────────────────
# Chunker
# ─────────────────────────────────────────────────────

class Chunker:
    """Split documents into chunks for embedding."""
    
    def __init__(self, chunk_size: int = 1000, chunk_overlap: int = 200):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.separators = ["\n\n", "\n", ". ", " "]
    
    def chunk_document(self, document: Dict) -> List[Dict]:
        """Split a document into chunks, preserving metadata."""
        text = document["text"]
        chunks = self._recursive_split(text)
        
        return [
            {
                "text": chunk,
                "source": document["source"],
                "filename": document["filename"],
                "chunk_index": i,
                "total_chunks": len(chunks)
            }
            for i, chunk in enumerate(chunks)
        ]
    
    def _recursive_split(self, text: str) -> List[str]:
        """Recursively split text respecting sentence boundaries."""
        if len(text) <= self.chunk_size:
            return [text] if text.strip() else []
        
        # Find best separator
        for sep in self.separators:
            if sep in text:
                splits = text.split(sep)
                break
        else:
            # No separator found, force split
            return [text[:self.chunk_size], text[self.chunk_size:]]
        
        # Merge splits into appropriate chunks
        chunks = []
        current = ""
        
        for split in splits:
            candidate = current + sep + split if current else split
            if len(candidate) <= self.chunk_size:
                current = candidate
            else:
                if current:
                    chunks.append(current)
                current = split
        
        if current:
            chunks.append(current)
        
        return [c.strip() for c in chunks if c.strip()]


# ─────────────────────────────────────────────────────
# Vector Store
# ─────────────────────────────────────────────────────

class VectorStore:
    """Manage vector storage and retrieval."""
    
    def __init__(self, config: RAGConfig):
        self.config = config
        self.client = chromadb.PersistentClient(path=config.db_path)
        self.embedding_fn = embedding_functions.SentenceTransformerEmbeddingFunction(
            model_name=config.embedding_model
        )
        self.collection = self.client.get_or_create_collection(
            name=config.collection_name,
            embedding_function=self.embedding_fn,
            metadata={"hnsw:space": "cosine"}
        )
    
    def add_chunks(self, chunks: List[Dict]):
        """Add document chunks to the vector store."""
        # ChromaDB requires unique IDs
        ids = [f"{c['filename']}_{c['chunk_index']}" for c in chunks]
        documents = [c["text"] for c in chunks]
        metadatas = [{
            "source": c["source"],
            "filename": c["filename"],
            "chunk_index": c["chunk_index"]
        } for c in chunks]
        
        # Add in batches (ChromaDB has batch limits)
        batch_size = 100
        for i in range(0, len(ids), batch_size):
            self.collection.add(
                ids=ids[i:i+batch_size],
                documents=documents[i:i+batch_size],
                metadatas=metadatas[i:i+batch_size]
            )
        
        print(f"Added {len(chunks)} chunks to vector store")
    
    def search(self, query: str, top_k: int = None, filter_metadata: dict = None) -> List[Dict]:
        """Search for relevant chunks."""
        k = top_k or self.config.top_k
        
        query_params = {
            "query_texts": [query],
            "n_results": k,
            "include": ["documents", "metadatas", "distances"]
        }
        
        if filter_metadata:
            query_params["where"] = filter_metadata
        
        results = self.collection.query(**query_params)
        
        # Format results
        retrieved = []
        for doc, meta, dist in zip(
            results["documents"][0],
            results["metadatas"][0],
            results["distances"][0]
        ):
            retrieved.append({
                "text": doc,
                "metadata": meta,
                "score": 1 - dist  # Convert distance to similarity
            })
        
        return retrieved


# ─────────────────────────────────────────────────────
# RAG Chain (The main pipeline)
# ─────────────────────────────────────────────────────

class RAGPipeline:
    """Complete RAG pipeline: Query → Retrieve → Augment → Generate."""
    
    def __init__(self, config: RAGConfig = None):
        self.config = config or RAGConfig()
        self.vector_store = VectorStore(self.config)
        self.llm_client = OpenAI()
        self.chunker = Chunker(
            chunk_size=self.config.chunk_size,
            chunk_overlap=self.config.chunk_overlap
        )
    
    def ingest(self, documents: List[Dict]):
        """Ingest documents into the RAG system."""
        all_chunks = []
        for doc in documents:
            chunks = self.chunker.chunk_document(doc)
            all_chunks.extend(chunks)
        
        self.vector_store.add_chunks(all_chunks)
        print(f"Ingested {len(documents)} documents → {len(all_chunks)} chunks")
    
    def query(self, question: str, filter_metadata: dict = None) -> Dict:
        """
        Answer a question using RAG.
        Returns the answer and source documents.
        """
        # Step 1: Retrieve relevant chunks
        retrieved = self.vector_store.search(
            query=question,
            filter_metadata=filter_metadata
        )
        
        # Step 2: Build augmented prompt
        context = "\n\n---\n\n".join([
            f"[Source: {r['metadata']['filename']}]\n{r['text']}"
            for r in retrieved
        ])
        
        prompt = self._build_prompt(question, context)
        
        # Step 3: Generate answer with LLM
        response = self.llm_client.chat.completions.create(
            model=self.config.llm_model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "You are a helpful assistant that answers questions based on "
                        "the provided context. If the context doesn't contain enough "
                        "information to answer, say so. Always cite your sources."
                    )
                },
                {"role": "user", "content": prompt}
            ],
            temperature=self.config.temperature
        )
        
        answer = response.choices[0].message.content
        
        return {
            "question": question,
            "answer": answer,
            "sources": retrieved,
            "context_used": context
        }
    
    def _build_prompt(self, question: str, context: str) -> str:
        """Build the augmented prompt."""
        return f"""Answer the following question based on the provided context.
If the context doesn't contain relevant information, say "I don't have enough information to answer this."

Context:
{context}

Question: {question}

Answer:"""


# ═══════════════════════════════════════════════════════
# USAGE
# ═══════════════════════════════════════════════════════

# Initialize
config = RAGConfig(
    chunk_size=800,
    chunk_overlap=100,
    top_k=5,
    llm_model="gpt-4"
)
rag = RAGPipeline(config)

# Ingest documents
docs = DocumentLoader.load_directory("./knowledge_base/")
rag.ingest(docs)

# Query
result = rag.query("What is our refund policy for electronics?")
print(f"Answer: {result['answer']}")
print(f"\nSources used:")
for source in result['sources']:
    print(f"  - {source['metadata']['filename']} (score: {source['score']:.3f})")
```

---

## Advanced RAG Techniques

### 1. Hybrid Search (Dense + Sparse)

Combine semantic search (embeddings) with keyword search (BM25) for better retrieval:

```python
# Hybrid Search: Best of both worlds
from rank_bm25 import BM25Okapi
import numpy as np

class HybridRetriever:
    """Combines dense (semantic) and sparse (BM25) retrieval."""
    
    def __init__(self, documents: List[str], embeddings: np.ndarray,
                 embed_model: SentenceTransformer, alpha: float = 0.7):
        self.documents = documents
        self.embeddings = embeddings
        self.embed_model = embed_model
        self.alpha = alpha  # Weight for dense search (1-alpha for sparse)
        
        # Initialize BM25
        tokenized_docs = [doc.lower().split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized_docs)
    
    def search(self, query: str, top_k: int = 5) -> List[Dict]:
        """Hybrid search combining dense and sparse retrieval."""
        
        # Dense search (semantic similarity)
        query_embedding = self.embed_model.encode([query])
        dense_scores = np.dot(self.embeddings, query_embedding.T).flatten()
        # Normalize to [0, 1]
        dense_scores = (dense_scores - dense_scores.min()) / (dense_scores.max() - dense_scores.min() + 1e-8)
        
        # Sparse search (BM25 keyword matching)
        sparse_scores = self.bm25.get_scores(query.lower().split())
        # Normalize to [0, 1]
        sparse_scores = (sparse_scores - sparse_scores.min()) / (sparse_scores.max() - sparse_scores.min() + 1e-8)
        
        # Combine scores
        hybrid_scores = self.alpha * dense_scores + (1 - self.alpha) * sparse_scores
        
        # Get top-k
        top_indices = np.argsort(hybrid_scores)[::-1][:top_k]
        
        results = []
        for idx in top_indices:
            results.append({
                "text": self.documents[idx],
                "score": hybrid_scores[idx],
                "dense_score": dense_scores[idx],
                "sparse_score": sparse_scores[idx]
            })
        
        return results
```

### 2. Re-Ranking

After initial retrieval, use a cross-encoder to re-rank results for better precision:

```python
# Re-ranking with Cross-Encoder
from sentence_transformers import CrossEncoder

class Reranker:
    """Re-rank retrieved documents using a cross-encoder."""
    
    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-12-v2"):
        self.model = CrossEncoder(model_name)
    
    def rerank(self, query: str, documents: List[str], top_k: int = 3) -> List[Dict]:
        """
        Re-rank documents based on relevance to query.
        Cross-encoders are more accurate but slower than bi-encoders.
        """
        # Cross-encoder takes (query, document) pairs
        pairs = [[query, doc] for doc in documents]
        
        # Score each pair
        scores = self.model.predict(pairs)
        
        # Sort by score
        ranked = sorted(
            zip(documents, scores),
            key=lambda x: x[1],
            reverse=True
        )
        
        return [
            {"text": doc, "score": float(score)}
            for doc, score in ranked[:top_k]
        ]

# Usage in RAG pipeline:
# 1. Retrieve top 20 with fast bi-encoder
# 2. Re-rank to top 5 with accurate cross-encoder
# This gives you both speed AND accuracy
```

### 3. Query Transformation

```python
# Query Expansion / Decomposition
def expand_query(query: str, llm_client) -> List[str]:
    """
    Generate multiple search queries from a single user question.
    Helps retrieve more relevant documents.
    """
    response = llm_client.chat.completions.create(
        model="gpt-4",
        messages=[{
            "role": "user",
            "content": f"""Generate 3 different search queries that would help answer 
            the following question. Return only the queries, one per line.
            
            Question: {query}"""
        }],
        temperature=0.7
    )
    
    queries = response.choices[0].message.content.strip().split("\n")
    queries.append(query)  # Include original query
    return queries

# HyDE: Hypothetical Document Embeddings
def hyde_query(query: str, llm_client, embed_model) -> np.ndarray:
    """
    Generate a hypothetical answer, then use its embedding for search.
    Often retrieves better documents than the raw question embedding.
    """
    # Generate hypothetical answer
    response = llm_client.chat.completions.create(
        model="gpt-4",
        messages=[{
            "role": "user",
            "content": f"Answer this question in a detailed paragraph: {query}"
        }],
        temperature=0.7
    )
    
    hypothetical_answer = response.choices[0].message.content
    
    # Embed the hypothetical answer (not the question!)
    embedding = embed_model.encode([hypothetical_answer])
    return embedding
```

### 4. Multi-Step RAG (Agentic RAG)

```python
# Iterative RAG: Ask follow-up questions if first retrieval isn't enough
def iterative_rag(question: str, rag_pipeline, max_iterations: int = 3) -> str:
    """
    Multi-step RAG that refines its search based on initial results.
    """
    all_context = []
    current_query = question
    
    for i in range(max_iterations):
        # Retrieve
        results = rag_pipeline.vector_store.search(current_query, top_k=3)
        all_context.extend([r["text"] for r in results])
        
        # Check if we have enough context
        check_prompt = f"""Given the following context, can you fully answer this question?
        Question: {question}
        Context: {' '.join(all_context[-3:])}
        
        Reply with ONLY 'YES' or 'NO: [what information is missing]'"""
        
        check = rag_pipeline.llm_client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": check_prompt}],
            temperature=0
        )
        
        response = check.choices[0].message.content
        if response.startswith("YES"):
            break
        
        # Generate refined query based on what's missing
        missing = response.replace("NO:", "").strip()
        current_query = f"{question} - specifically: {missing}"
    
    # Final answer with all gathered context
    context = "\n\n".join(all_context)
    # ... generate final answer with full context
```

### 5. Contextual Compression

```python
# Extract only the relevant portions from retrieved chunks
def compress_context(query: str, documents: List[str], llm_client) -> List[str]:
    """
    Compress retrieved documents to only include relevant information.
    Reduces noise and fits more useful context in the prompt.
    """
    compressed = []
    
    for doc in documents:
        response = llm_client.chat.completions.create(
            model="gpt-4",
            messages=[{
                "role": "user",
                "content": f"""Extract ONLY the sentences from the following text that are 
                relevant to answering: "{query}"
                
                If nothing is relevant, respond with "NOT_RELEVANT".
                
                Text: {doc}"""
            }],
            temperature=0
        )
        
        result = response.choices[0].message.content
        if result != "NOT_RELEVANT":
            compressed.append(result)
    
    return compressed
```

---

## Evaluation and Metrics

### RAG Evaluation Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                RAG EVALUATION DIMENSIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. RETRIEVAL QUALITY                                            │
│     ├── Precision@K: What % of retrieved docs are relevant?      │
│     ├── Recall@K: What % of relevant docs were retrieved?        │
│     ├── MRR: How high is the first relevant result ranked?       │
│     └── NDCG: Overall ranking quality                            │
│                                                                  │
│  2. GENERATION QUALITY                                           │
│     ├── Faithfulness: Does answer only use info from context?    │
│     ├── Relevance: Does answer actually address the question?    │
│     ├── Completeness: Does answer cover all aspects?             │
│     └── Correctness: Is the answer factually correct?            │
│                                                                  │
│  3. END-TO-END QUALITY                                           │
│     ├── Answer accuracy vs ground truth                          │
│     ├── User satisfaction (A/B testing)                          │
│     └── Latency and cost                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Code: RAG Evaluation with RAGAS

```python
# RAGAS: Standard framework for RAG evaluation
# pip install ragas

from ragas import evaluate
from ragas.metrics import (
    faithfulness,        # Does answer stick to context?
    answer_relevancy,    # Is answer relevant to question?
    context_precision,   # Are retrieved docs relevant?
    context_recall       # Did we retrieve all needed docs?
)
from datasets import Dataset

# Prepare evaluation dataset
eval_data = {
    "question": [
        "What is our return policy?",
        "How do I reset my password?"
    ],
    "answer": [  # Generated by your RAG system
        "You can return items within 30 days with receipt.",
        "Click 'Forgot Password' on the login page."
    ],
    "contexts": [  # Retrieved chunks
        [["Returns accepted within 30 days. Original receipt required."]],
        [["To reset password, use the Forgot Password link on login."]]
    ],
    "ground_truth": [  # Human-written correct answers
        "Items can be returned within 30 days with original receipt.",
        "Use the Forgot Password link on the login page to reset."
    ]
}

dataset = Dataset.from_dict(eval_data)

# Run evaluation
results = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall]
)

print(results)
# Output: {'faithfulness': 0.95, 'answer_relevancy': 0.88, 
#           'context_precision': 0.90, 'context_recall': 0.85}
```

---

## Common Mistakes

### 1. Chunks Too Large or Too Small

**Mistake**: Using 5000-character chunks (too vague) or 100-character chunks (no context).
**Fix**: Start with 500-1000 characters with 10-20% overlap. Test and iterate.

### 2. No Metadata Filtering

**Mistake**: Searching all documents when the user asks about a specific topic/time.
**Fix**: Store metadata (date, source, category) and filter before/during search.

### 3. Ignoring the Embedding Model's Token Limit

**Mistake**: Passing 2000-token chunks to a model with a 256-token limit → silent truncation.
**Fix**: Always check your embedding model's max tokens. Chunk accordingly.

### 4. Not Handling "No Relevant Results"

**Mistake**: Always stuffing retrieved context even when it's irrelevant (low scores).
**Fix**: Set a similarity threshold. If no results pass the threshold, tell the user you don't know.

```python
# Set a minimum relevance threshold
MIN_SCORE = 0.3  # Adjust based on your data

results = vector_store.search(query)
relevant = [r for r in results if r["score"] > MIN_SCORE]

if not relevant:
    return "I don't have information about that in my knowledge base."
```

### 5. Same Embedding Model for Different Content Types

**Mistake**: Using a general-purpose embedding model for code, tables, or highly technical content.
**Fix**: Use domain-specific models (e.g., `codellama` for code) or multi-modal embeddings for mixed content.

### 6. Not Updating the Knowledge Base

**Mistake**: Indexing once and never refreshing.
**Fix**: Implement incremental updates with change detection and re-indexing schedules.

### 7. Prompt Doesn't Instruct How to Use Context

**Mistake**: Just appending context without instructions.
**Fix**: Explicitly tell the LLM to answer based on context, cite sources, and say when it doesn't know.

---

## Interview Questions

### Conceptual

**Q1: Explain RAG and why it's better than just increasing context length.**
> RAG retrieves only relevant information, reducing noise, cost, and latency. Even with 128K context models, you can't fit millions of documents. RAG scales to unlimited knowledge while keeping prompts focused.

**Q2: What are the tradeoffs between dense (embedding) and sparse (BM25) retrieval?**
> Dense captures semantic meaning ("happy" ≈ "joyful") but can miss exact keywords. Sparse excels at exact matches and rare terms but misses paraphrases. Hybrid combines both for best results.

**Q3: How do you handle documents that are updated frequently?**
> Implement incremental indexing: track document versions, delete old chunks, re-embed updated docs. Use metadata timestamps for freshness filtering. Consider CDC (Change Data Capture) patterns.

**Q4: What is the "Lost in the Middle" problem?**
> LLMs pay more attention to the beginning and end of long contexts, often missing information in the middle. Mitigation: put most relevant chunks first, use re-ranking, limit context length, or use recursive summarization.

**Q5: How would you evaluate a RAG system in production?**
> Track: retrieval precision/recall (are we finding the right docs?), faithfulness (does the answer match the context?), answer relevance (does it answer the question?), latency, user feedback (thumbs up/down), and compare against baseline (no-RAG) answers.

### Practical/System Design

**Q6: Design a RAG system for a company with 10,000 internal documents.**
> Document ingestion pipeline → Recursive chunking (800 tokens, 100 overlap) → Embed with `text-embedding-3-small` → Store in Qdrant/Pinecone → Query: embed question → retrieve top 10 → re-rank to top 3 → augment prompt → GPT-4 generates answer with citations. Add: metadata filtering, access control, feedback loop, monitoring dashboard.

**Q7: Your RAG system retrieves relevant documents but the LLM still hallucinates. Why?**
> Causes: (1) Context is relevant but doesn't directly answer the question, (2) LLM ignores context and uses parametric knowledge, (3) Conflicting information in retrieved docs. Fix: stronger system prompt about sticking to context, add "if not in context, say I don't know", use faithfulness evaluation, consider fine-tuning the LLM to follow context better.

**Q8: How do you handle multi-hop questions in RAG?**
> Multi-hop requires info from multiple documents. Solutions: (1) Query decomposition — break into sub-questions, (2) Iterative retrieval — retrieve, reason, retrieve more, (3) Graph RAG — use knowledge graphs to find connections, (4) Agentic RAG — let an agent plan retrieval steps.

---

## Quick Reference

### RAG Pipeline Checklist

```
┌─────────────────────────────────────────────────────┐
│          RAG IMPLEMENTATION CHECKLIST                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  □ Choose embedding model (match to your domain)    │
│  □ Design chunking strategy (test different sizes!) │
│  □ Select vector database (scale needs)             │
│  □ Implement document loader (all your formats)     │
│  □ Add metadata to chunks (source, date, category)  │
│  □ Set similarity threshold (don't force bad results)│
│  □ Write good system prompt (cite sources, etc.)    │
│  □ Add re-ranking (if precision matters)            │
│  □ Implement evaluation pipeline                    │
│  □ Plan for updates/re-indexing                     │
│  □ Monitor in production (latency, quality, cost)   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Key Formulas

| Metric | Formula | Range |
|--------|---------|-------|
| Cosine Similarity | $\frac{\vec{a} \cdot \vec{b}}{‖\vec{a}‖ \times ‖\vec{b}‖}$ | [-1, 1] |
| Precision@K | $\frac{\text{relevant in top K}}{K}$ | [0, 1] |
| Recall@K | $\frac{\text{relevant in top K}}{\text{total relevant}}$ | [0, 1] |
| MRR | $\frac{1}{\text{rank of first relevant}}$ | (0, 1] |
| F1 | $\frac{2 \times P \times R}{P + R}$ | [0, 1] |

### Technology Selection Guide

| Need | Recommended Stack |
|------|-------------------|
| **Prototype** | ChromaDB + Sentence-Transformers + OpenAI |
| **Production (small)** | Qdrant + text-embedding-3-small + GPT-4 |
| **Production (large)** | Pinecone/Weaviate + text-embedding-3-large + GPT-4 |
| **On-premise (secure)** | FAISS/Milvus + BGE/E5 + LLaMA/Mistral |
| **Existing Postgres** | pgvector + OpenAI embeddings + GPT-4 |

---

*Next Chapter: [06-LangChain-and-Agents](06-LangChain-and-Agents.md)*
