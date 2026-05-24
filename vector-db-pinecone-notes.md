# Vector Databases & Pinecone Notes

---

## Table of Contents

1. [Pinecone Raw REST API (Postman)](#1-pinecone-raw-rest-api-postman)
   - [1.1 Overview & Setup](#11-overview--setup)
   - [1.2 Authentication](#12-authentication)
   - [1.3 Index Operations](#13-index-operations)
   - [1.4 Vector Operations](#14-vector-operations)
   - [1.5 Collection Operations](#15-collection-operations)
2. [Pinecone Python SDK](#2-pinecone-python-sdk)
   - [2.1 Installation & Setup](#21-installation--setup)
   - [2.2 Index Management](#22-index-management)
   - [2.3 Vector Operations](#23-vector-operations)
   - [2.4 Namespaces](#24-namespaces)
   - [2.5 Metadata Filtering](#25-metadata-filtering)
   - [2.6 Sparse-Dense (Hybrid) Search](#26-sparse-dense-hybrid-search)
3. [Pinecone with LangChain](#3-pinecone-with-langchain)
   - [3.1 Installation & Setup](#31-installation--setup)
   - [3.2 Document Ingestion](#32-document-ingestion)
   - [3.3 Similarity Search](#33-similarity-search)
   - [3.4 Using as a Retriever in RAG Chain](#34-using-as-a-retriever-in-rag-chain)
   - [3.5 Metadata Filtering with LangChain](#35-metadata-filtering-with-langchain)
4. [Other Vector Databases with LangChain](#4-other-vector-databases-with-langchain)
   - [4.1 Chroma](#41-chroma)
   - [4.2 FAISS](#42-faiss)
   - [4.3 Weaviate](#43-weaviate)
   - [4.4 Milvus / Zilliz](#44-milvus--zilliz)
   - [4.5 Qdrant](#45-qdrant)
   - [4.6 PGVector (PostgreSQL)](#46-pgvector-postgresql)
   - [4.7 Redis Vector Search](#47-redis-vector-search)
   - [4.8 Comparison Table](#48-comparison-table)

---

## 1. Pinecone Raw REST API (Postman)

### 1.1 Overview & Setup

- Pinecone is a managed vector database for similarity search at scale.
- Offers a REST API accessible via Postman, curl, or any HTTP client.
- **Base URLs:**
  - **Control Plane (index management):** `https://api.pinecone.io`
  - **Data Plane (vector ops):** `https://<index-name>-<project-id>.svc.<environment>.pinecone.io`
- Sign up at [https://www.pinecone.io](https://www.pinecone.io) and get your API key from the dashboard.

### 1.2 Authentication

All requests require the API key in the header:

```
Header: Api-Key: YOUR_API_KEY
Content-Type: application/json
```

### 1.3 Index Operations

#### List Indexes

```
GET https://api.pinecone.io/indexes
Headers:
  Api-Key: YOUR_API_KEY
```

**Response:**

```json
{
  "indexes": [
    {
      "name": "my-index",
      "dimension": 1536,
      "metric": "cosine",
      "host": "my-index-abc123.svc.us-east-1-aws.pinecone.io",
      "status": { "ready": true, "state": "Ready" }
    }
  ]
}
```

#### Create Index (Serverless)

```
POST https://api.pinecone.io/indexes
Headers:
  Api-Key: YOUR_API_KEY
  Content-Type: application/json

Body:
{
  "name": "my-index",
  "dimension": 1536,
  "metric": "cosine",
  "spec": {
    "serverless": {
      "cloud": "aws",
      "region": "us-east-1"
    }
  }
}
```

> **Metrics:** `cosine`, `euclidean`, `dotproduct`

#### Create Index (Pod-Based)

```
POST https://api.pinecone.io/indexes
Body:
{
  "name": "my-pod-index",
  "dimension": 1536,
  "metric": "cosine",
  "spec": {
    "pod": {
      "environment": "us-east-1-aws",
      "pod_type": "p1.x1",
      "pods": 1,
      "replicas": 1
    }
  }
}
```

#### Describe Index

```
GET https://api.pinecone.io/indexes/my-index
Headers:
  Api-Key: YOUR_API_KEY
```

#### Delete Index

```
DELETE https://api.pinecone.io/indexes/my-index
Headers:
  Api-Key: YOUR_API_KEY
```

#### Configure Index (Scale Replicas)

```
PATCH https://api.pinecone.io/indexes/my-index
Body:
{
  "spec": {
    "pod": {
      "replicas": 2
    }
  }
}
```

### 1.4 Vector Operations

> **Data Plane Base URL:** Use the `host` value from the Describe Index response.
> Example: `https://my-index-abc123.svc.us-east-1-aws.pinecone.io`

#### Upsert Vectors

```
POST https://{index-host}/vectors/upsert
Headers:
  Api-Key: YOUR_API_KEY
  Content-Type: application/json

Body:
{
  "vectors": [
    {
      "id": "vec-001",
      "values": [0.1, 0.2, 0.3, ...],
      "metadata": {
        "genre": "comedy",
        "year": 2020
      }
    },
    {
      "id": "vec-002",
      "values": [0.4, 0.5, 0.6, ...],
      "metadata": {
        "genre": "drama",
        "year": 2021
      }
    }
  ],
  "namespace": "my-namespace"
}
```

**Response:**

```json
{ "upsertedCount": 2 }
```

#### Query (Similarity Search)

```
POST https://{index-host}/query
Body:
{
  "vector": [0.1, 0.2, 0.3, ...],
  "topK": 5,
  "includeMetadata": true,
  "includeValues": false,
  "namespace": "my-namespace",
  "filter": {
    "genre": { "$eq": "comedy" },
    "year": { "$gte": 2020 }
  }
}
```

**Response:**

```json
{
  "matches": [
    {
      "id": "vec-001",
      "score": 0.95,
      "metadata": { "genre": "comedy", "year": 2020 }
    }
  ],
  "namespace": "my-namespace"
}
```

#### Query by ID

```
POST https://{index-host}/query
Body:
{
  "id": "vec-001",
  "topK": 5,
  "includeMetadata": true,
  "namespace": "my-namespace"
}
```

#### Fetch Vectors by ID

```
GET https://{index-host}/vectors/fetch?ids=vec-001&ids=vec-002&namespace=my-namespace
Headers:
  Api-Key: YOUR_API_KEY
```

#### Update Vector Metadata

```
POST https://{index-host}/vectors/update
Body:
{
  "id": "vec-001",
  "setMetadata": {
    "genre": "action"
  },
  "namespace": "my-namespace"
}
```

#### Delete Vectors

```
POST https://{index-host}/vectors/delete
Body:
{
  "ids": ["vec-001", "vec-002"],
  "namespace": "my-namespace"
}
```

#### Delete All Vectors in a Namespace

```
POST https://{index-host}/vectors/delete
Body:
{
  "deleteAll": true,
  "namespace": "my-namespace"
}
```

#### Describe Index Stats

```
POST https://{index-host}/describe_index_stats
Body: {}
```

**Response:**

```json
{
  "dimension": 1536,
  "indexFullness": 0.05,
  "totalVectorCount": 10000,
  "namespaces": {
    "my-namespace": { "vectorCount": 5000 },
    "": { "vectorCount": 5000 }
  }
}
```

### 1.5 Collection Operations

#### Create Collection (Backup an Index)

```
POST https://api.pinecone.io/collections
Body:
{
  "name": "my-collection",
  "source": "my-index"
}
```

#### List Collections

```
GET https://api.pinecone.io/collections
```

#### Describe Collection

```
GET https://api.pinecone.io/collections/my-collection
```

#### Delete Collection

```
DELETE https://api.pinecone.io/collections/my-collection
```

#### Create Index from Collection

```
POST https://api.pinecone.io/indexes
Body:
{
  "name": "restored-index",
  "dimension": 1536,
  "metric": "cosine",
  "spec": {
    "pod": {
      "environment": "us-east-1-aws",
      "pod_type": "p1.x1",
      "pods": 1,
      "sourceCollection": "my-collection"
    }
  }
}
```

#### Filter Operators Reference

| Operator | Description             | Example                            |
|----------|-------------------------|------------------------------------|
| `$eq`    | Equal                   | `{"genre": {"$eq": "comedy"}}`     |
| `$ne`    | Not equal               | `{"genre": {"$ne": "drama"}}`      |
| `$gt`    | Greater than            | `{"year": {"$gt": 2020}}`          |
| `$gte`   | Greater than or equal   | `{"year": {"$gte": 2020}}`         |
| `$lt`    | Less than               | `{"year": {"$lt": 2025}}`          |
| `$lte`   | Less than or equal      | `{"year": {"$lte": 2025}}`         |
| `$in`    | In array                | `{"genre": {"$in": ["a","b"]}}`    |
| `$nin`   | Not in array            | `{"genre": {"$nin": ["a","b"]}}`   |
| `$and`   | Logical AND             | `{"$and": [{...}, {...}]}`         |
| `$or`    | Logical OR              | `{"$or": [{...}, {...}]}`          |

---

## 2. Pinecone Python SDK

### 2.1 Installation & Setup

```bash
pip install pinecone
```

```python
from pinecone import Pinecone, ServerlessSpec, PodSpec

# Initialize client
pc = Pinecone(api_key="YOUR_API_KEY")

# Or use environment variable PINECONE_API_KEY
# export PINECONE_API_KEY="your-key"
pc = Pinecone()  # auto-reads from env
```

### 2.2 Index Management

#### Create Serverless Index

```python
pc.create_index(
    name="my-index",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(
        cloud="aws",
        region="us-east-1"
    )
)
```

#### Create Pod-Based Index

```python
pc.create_index(
    name="my-pod-index",
    dimension=1536,
    metric="cosine",
    spec=PodSpec(
        environment="us-east-1-aws",
        pod_type="p1.x1",
        pods=1
    )
)
```

#### Wait for Index to be Ready

```python
import time

while not pc.describe_index("my-index").status["ready"]:
    time.sleep(1)
```

#### List, Describe, Delete Indexes

```python
# List all indexes
indexes = pc.list_indexes()
for idx in indexes:
    print(idx.name, idx.dimension, idx.metric)

# Describe an index
desc = pc.describe_index("my-index")
print(desc.host, desc.status)

# Delete an index
pc.delete_index("my-index")
```

#### Connect to an Index

```python
index = pc.Index("my-index")

# Or connect by host directly
index = pc.Index(host="my-index-abc123.svc.us-east-1-aws.pinecone.io")
```

### 2.3 Vector Operations

#### Upsert

```python
index.upsert(
    vectors=[
        {
            "id": "vec-001",
            "values": [0.1, 0.2, 0.3, ...],
            "metadata": {"genre": "comedy", "year": 2020}
        },
        {
            "id": "vec-002",
            "values": [0.4, 0.5, 0.6, ...],
            "metadata": {"genre": "drama", "year": 2021}
        }
    ],
    namespace="my-namespace"
)
```

#### Upsert in Batches

```python
from itertools import islice

def chunked(iterable, batch_size=100):
    it = iter(iterable)
    chunk = list(islice(it, batch_size))
    while chunk:
        yield chunk
        chunk = list(islice(it, batch_size))

for batch in chunked(all_vectors, batch_size=100):
    index.upsert(vectors=batch, namespace="my-namespace")
```

#### Query

```python
results = index.query(
    vector=[0.1, 0.2, 0.3, ...],
    top_k=5,
    include_metadata=True,
    include_values=False,
    namespace="my-namespace",
    filter={
        "genre": {"$eq": "comedy"},
        "year": {"$gte": 2020}
    }
)

for match in results["matches"]:
    print(match["id"], match["score"], match["metadata"])
```

#### Query by ID

```python
results = index.query(
    id="vec-001",
    top_k=5,
    include_metadata=True,
    namespace="my-namespace"
)
```

#### Fetch

```python
fetched = index.fetch(
    ids=["vec-001", "vec-002"],
    namespace="my-namespace"
)

for id, vec in fetched["vectors"].items():
    print(id, vec["metadata"])
```

#### Update

```python
index.update(
    id="vec-001",
    set_metadata={"genre": "action"},
    namespace="my-namespace"
)
```

#### Delete

```python
# Delete by IDs
index.delete(ids=["vec-001", "vec-002"], namespace="my-namespace")

# Delete by metadata filter
index.delete(
    filter={"genre": {"$eq": "comedy"}},
    namespace="my-namespace"
)

# Delete all in a namespace
index.delete(delete_all=True, namespace="my-namespace")
```

#### Index Stats

```python
stats = index.describe_index_stats()
print(stats.total_vector_count)
print(stats.dimension)
print(stats.namespaces)
```

### 2.4 Namespaces

- Namespaces partition vectors within an index.
- Queries are scoped to a single namespace.
- Default namespace is `""` (empty string).

```python
# Upsert into a namespace
index.upsert(vectors=[...], namespace="products")

# Query within a namespace
index.query(vector=[...], top_k=5, namespace="products")

# Delete all vectors in a namespace
index.delete(delete_all=True, namespace="products")
```

### 2.5 Metadata Filtering

```python
# Combined filter
results = index.query(
    vector=[0.1, 0.2, ...],
    top_k=10,
    filter={
        "$and": [
            {"genre": {"$in": ["comedy", "action"]}},
            {"year": {"$gte": 2019}},
            {"year": {"$lte": 2023}}
        ]
    },
    include_metadata=True
)
```

### 2.6 Sparse-Dense (Hybrid) Search

```python
from pinecone import SparseValues

# Upsert with sparse values
index.upsert(
    vectors=[
        {
            "id": "vec-001",
            "values": [0.1, 0.2, ...],           # dense
            "sparse_values": {
                "indices": [10, 50, 100],
                "values": [0.5, 0.3, 0.8]
            },
            "metadata": {"text": "hello world"}
        }
    ]
)

# Hybrid query
results = index.query(
    vector=[0.1, 0.2, ...],
    sparse_vector={
        "indices": [10, 50],
        "values": [0.5, 0.3]
    },
    top_k=10,
    include_metadata=True
)
```

---

## 3. Pinecone with LangChain

### 3.1 Installation & Setup

```bash
pip install langchain langchain-pinecone langchain-openai pinecone
```

```python
import os
os.environ["PINECONE_API_KEY"] = "your-pinecone-api-key"
os.environ["OPENAI_API_KEY"] = "your-openai-api-key"

from pinecone import Pinecone, ServerlessSpec
from langchain_pinecone import PineconeVectorStore
from langchain_openai import OpenAIEmbeddings

# Initialize Pinecone
pc = Pinecone()

# Create index if it doesn't exist
index_name = "langchain-index"
if index_name not in [idx.name for idx in pc.list_indexes()]:
    pc.create_index(
        name=index_name,
        dimension=1536,
        metric="cosine",
        spec=ServerlessSpec(cloud="aws", region="us-east-1")
    )

# Embeddings model
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
```

### 3.2 Document Ingestion

#### From Raw Texts

```python
from langchain_core.documents import Document

docs = [
    Document(page_content="AI is transforming healthcare.", metadata={"source": "article1", "topic": "AI"}),
    Document(page_content="Vector databases store embeddings.", metadata={"source": "article2", "topic": "DB"}),
    Document(page_content="LangChain simplifies LLM apps.", metadata={"source": "article3", "topic": "AI"}),
]

vectorstore = PineconeVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    index_name=index_name,
    namespace="articles"
)
```

#### From Files (PDF, Text, etc.)

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

# Load
loader = PyPDFLoader("document.pdf")
pages = loader.load()

# Split
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = splitter.split_documents(pages)

# Ingest
vectorstore = PineconeVectorStore.from_documents(
    documents=chunks,
    embedding=embeddings,
    index_name=index_name,
    namespace="pdfs"
)
```

#### Connect to Existing Index

```python
vectorstore = PineconeVectorStore(
    index_name=index_name,
    embedding=embeddings,
    namespace="articles"
)
```

#### Add Documents to Existing Store

```python
new_docs = [
    Document(page_content="New content here.", metadata={"source": "article4"})
]
vectorstore.add_documents(new_docs)
```

### 3.3 Similarity Search

```python
# Basic similarity search
results = vectorstore.similarity_search(
    query="What is AI?",
    k=3
)
for doc in results:
    print(doc.page_content, doc.metadata)

# Similarity search with scores
results_with_scores = vectorstore.similarity_search_with_score(
    query="What is AI?",
    k=3
)
for doc, score in results_with_scores:
    print(f"Score: {score:.4f} | {doc.page_content}")

# MMR (Maximal Marginal Relevance) search — balances relevance and diversity
results = vectorstore.max_marginal_relevance_search(
    query="What is AI?",
    k=3,
    fetch_k=10
)
```

### 3.4 Using as a Retriever in RAG Chain

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# Create retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",   # or "mmr"
    search_kwargs={"k": 4}
)

# RAG prompt
template = """Answer the question based only on the context below.

Context:
{context}

Question: {question}
"""
prompt = ChatPromptTemplate.from_template(template)

# LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Build chain
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# Invoke
answer = rag_chain.invoke("How is AI used in healthcare?")
print(answer)
```

### 3.5 Metadata Filtering with LangChain

```python
# Filter during similarity search
results = vectorstore.similarity_search(
    query="AI tools",
    k=3,
    filter={"topic": "AI"}
)

# Filter in retriever
retriever = vectorstore.as_retriever(
    search_kwargs={
        "k": 5,
        "filter": {"topic": {"$eq": "AI"}}
    }
)
```

#### Delete Documents

```python
# Delete by IDs (returned during add_documents)
vectorstore.delete(ids=["id1", "id2"])
```

---

## 4. Other Vector Databases with LangChain

### 4.1 Chroma

**Open-source, lightweight, embeddable vector DB. Great for local dev and prototyping.**

```bash
pip install langchain-chroma chromadb
```

#### In-Memory (Ephemeral)

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    collection_name="my-collection"
)

results = vectorstore.similarity_search("query text", k=3)
```

#### Persistent (Local Disk)

```python
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    persist_directory="./chroma_db",
    collection_name="my-collection"
)

# Reconnect later
vectorstore = Chroma(
    persist_directory="./chroma_db",
    collection_name="my-collection",
    embedding_function=embeddings
)
```

#### Chroma with Client-Server Mode

```bash
# Run Chroma server
chroma run --host localhost --port 8000
```

```python
import chromadb

client = chromadb.HttpClient(host="localhost", port=8000)
vectorstore = Chroma(
    client=client,
    collection_name="my-collection",
    embedding_function=embeddings
)
```

#### Metadata Filtering

```python
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"source": "article1"}
)
```

---

### 4.2 FAISS

**Facebook AI Similarity Search. Local, fast, no server needed. Best for offline/batch workloads.**

```bash
pip install langchain-community faiss-cpu
# or for GPU: pip install faiss-gpu
```

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# Create from documents
vectorstore = FAISS.from_documents(documents=docs, embedding=embeddings)

# Search
results = vectorstore.similarity_search("query text", k=3)
results_with_scores = vectorstore.similarity_search_with_score("query text", k=3)

# MMR search
results = vectorstore.max_marginal_relevance_search("query", k=3, fetch_k=10)
```

#### Save & Load from Disk

```python
# Save
vectorstore.save_local("./faiss_index")

# Load
loaded_store = FAISS.load_local(
    "./faiss_index",
    embeddings,
    allow_dangerous_deserialization=True
)
```

#### Merge Two FAISS Indexes

```python
store1 = FAISS.from_documents(docs_batch_1, embeddings)
store2 = FAISS.from_documents(docs_batch_2, embeddings)
store1.merge_from(store2)
```

---

### 4.3 Weaviate

**Open-source, cloud-native vector DB with built-in vectorization and GraphQL API.**

```bash
pip install langchain-weaviate weaviate-client
```

#### Weaviate Cloud

```python
import weaviate
from weaviate.classes.init import Auth
from langchain_weaviate import WeaviateVectorStore
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# Connect to Weaviate Cloud
client = weaviate.connect_to_weaviate_cloud(
    cluster_url="https://your-cluster.weaviate.network",
    auth_credentials=Auth.api_key("your-weaviate-api-key")
)

vectorstore = WeaviateVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    client=client,
    index_name="LangChainDocs"
)

results = vectorstore.similarity_search("query text", k=3)

client.close()
```

#### Weaviate Local (Docker)

```bash
docker run -d -p 8080:8080 -p 50051:50051 semitechnologies/weaviate:latest
```

```python
client = weaviate.connect_to_local()

vectorstore = WeaviateVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    client=client,
    index_name="LangChainDocs"
)
```

---

### 4.4 Milvus / Zilliz

**Open-source (Milvus) and managed cloud (Zilliz). Built for billion-scale vector search.**

```bash
pip install langchain-milvus pymilvus
```

#### Milvus Lite (Local / Embedded)

```python
from langchain_milvus import Milvus
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

vectorstore = Milvus.from_documents(
    documents=docs,
    embedding=embeddings,
    connection_args={"uri": "./milvus_demo.db"},  # local file
    collection_name="langchain_docs"
)

results = vectorstore.similarity_search("query text", k=3)
```

#### Milvus Server (Docker)

```bash
docker compose up -d   # using milvus docker-compose
```

```python
vectorstore = Milvus.from_documents(
    documents=docs,
    embedding=embeddings,
    connection_args={"host": "localhost", "port": "19530"},
    collection_name="langchain_docs"
)
```

#### Zilliz Cloud (Managed)

```python
vectorstore = Milvus.from_documents(
    documents=docs,
    embedding=embeddings,
    connection_args={
        "uri": "https://your-instance.zillizcloud.com",
        "token": "your-zilliz-api-key"
    },
    collection_name="langchain_docs"
)
```

---

### 4.5 Qdrant

**High-performance, Rust-based vector DB with rich filtering and payload support.**

```bash
pip install langchain-qdrant qdrant-client
```

#### In-Memory

```python
from langchain_qdrant import QdrantVectorStore
from qdrant_client import QdrantClient
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

client = QdrantClient(":memory:")  # ephemeral

vectorstore = QdrantVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    client=client,
    collection_name="my_collection"
)

results = vectorstore.similarity_search("query text", k=3)
```

#### Persistent (Local Disk)

```python
client = QdrantClient(path="./qdrant_data")

vectorstore = QdrantVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    client=client,
    collection_name="my_collection"
)
```

#### Qdrant Server (Docker)

```bash
docker run -p 6333:6333 qdrant/qdrant
```

```python
client = QdrantClient(host="localhost", port=6333)

vectorstore = QdrantVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    client=client,
    collection_name="my_collection"
)
```

#### Qdrant Cloud

```python
client = QdrantClient(
    url="https://your-cluster.qdrant.io",
    api_key="your-qdrant-api-key"
)
```

#### Metadata Filtering

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue

results = vectorstore.similarity_search(
    "query",
    k=5,
    filter=Filter(
        must=[FieldCondition(key="metadata.topic", match=MatchValue(value="AI"))]
    )
)
```

---

### 4.6 PGVector (PostgreSQL)

**Vector search extension for PostgreSQL. Use your existing Postgres infra for vectors.**

```bash
pip install langchain-postgres psycopg[binary]
```

> Requires PostgreSQL with `pgvector` extension enabled.

```python
from langchain_postgres import PGVector
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

CONNECTION_STRING = "postgresql+psycopg://user:password@localhost:5432/mydb"

vectorstore = PGVector.from_documents(
    documents=docs,
    embedding=embeddings,
    connection=CONNECTION_STRING,
    collection_name="langchain_docs"
)

results = vectorstore.similarity_search("query text", k=3)
```

#### Existing Connection

```python
vectorstore = PGVector(
    connection=CONNECTION_STRING,
    collection_name="langchain_docs",
    embeddings=embeddings
)

# Add more documents
vectorstore.add_documents(new_docs)
```

---

### 4.7 Redis Vector Search

**Use Redis as a vector DB with the RediSearch module.**

```bash
pip install langchain-redis redis redisvl
```

> Requires Redis Stack (includes RediSearch): `docker run -p 6379:6379 redis/redis-stack:latest`

```python
from langchain_redis import RedisVectorStore, RedisConfig
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

config = RedisConfig(
    index_name="langchain_docs",
    redis_url="redis://localhost:6379",
    metadata_schema=[
        {"name": "source", "type": "tag"},
        {"name": "topic", "type": "tag"},
    ]
)

vectorstore = RedisVectorStore.from_documents(
    documents=docs,
    embedding=embeddings,
    config=config
)

results = vectorstore.similarity_search("query text", k=3)
```

---

### 4.8 Comparison Table

| Feature              | Pinecone        | Chroma         | FAISS          | Weaviate        | Milvus/Zilliz  | Qdrant          | PGVector       | Redis          |
|----------------------|-----------------|----------------|----------------|-----------------|----------------|-----------------|----------------|----------------|
| **Type**             | Managed cloud   | Open-source    | Library        | Open-source     | Open-source    | Open-source     | PG Extension   | In-memory DB   |
| **Hosting**          | Cloud only      | Local/Cloud    | Local only     | Local/Cloud     | Local/Cloud    | Local/Cloud     | Self/Cloud     | Self/Cloud     |
| **Scalability**      | High            | Medium         | Medium         | High            | Very High      | High            | Medium         | High           |
| **Setup Ease**       | Very Easy       | Very Easy      | Easy           | Medium          | Medium         | Easy            | Medium         | Medium         |
| **Metadata Filter**  | Yes             | Yes            | No (basic)     | Yes (GraphQL)   | Yes            | Yes (rich)      | Yes (SQL)      | Yes            |
| **Persistence**      | Cloud           | Local/Server   | File-based     | Server          | Server/File    | Server/File     | PostgreSQL     | RDB/AOF        |
| **Hybrid Search**    | Yes             | No             | No             | Yes (BM25)      | Yes            | Yes             | No             | Yes            |
| **Free Tier**        | Yes (limited)   | Unlimited      | Unlimited      | Yes (sandbox)   | Yes (Zilliz)   | Yes (1GB)       | N/A            | N/A            |
| **Best For**         | Production SaaS | Prototyping    | Offline/Batch  | GraphQL lovers  | Billion scale  | Rich filtering  | Existing PG    | Real-time apps |
| **LangChain Pkg**    | langchain-pinecone | langchain-chroma | langchain-community | langchain-weaviate | langchain-milvus | langchain-qdrant | langchain-postgres | langchain-redis |

---

> **Quick Tip:** For prototyping, use **Chroma** or **FAISS**. For production, evaluate **Pinecone**, **Qdrant**, or **Milvus** based on your scale and filtering needs. If you already use PostgreSQL, **PGVector** avoids adding new infrastructure.
