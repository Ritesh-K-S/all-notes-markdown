# 🔍 Chapter 3F.1 — Elasticsearch Architecture & Concepts

> **"Google indexes the internet. Elasticsearch lets YOU index ANYTHING — and search it in milliseconds."**

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 3A (NoSQL Foundations), basic JSON knowledge

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why** Elasticsearch exists and what problems it solves
- Know how the **Inverted Index** works (the secret sauce behind every search engine)
- Master the **cluster architecture** — Nodes, Shards, Replicas, and how they coordinate
- Understand **Analyzers, Tokenizers, and Mappings** — how text becomes searchable
- Know when to use Elasticsearch vs a traditional database
- Be able to design an Elasticsearch cluster for real-world workloads

---

## 🌍 Elasticsearch — The Origin Story (60 Seconds)

```
Born:        2010 (by Shay Banon)
Predecessor: Compass (Shay's earlier search library, built on Apache Lucene)
Built On:    Apache Lucene (the nuclear engine powering ALL serious search)
Written In:  Java
License:     SSPL / Elastic License 2.0 (since v7.11 — was Apache 2.0 before)
Powers:      GitHub, Wikipedia, Netflix, Uber, eBay, Shopify, Stack Overflow
Used For:    Full-text search, Log analytics, APM, Security analytics, Geo search
```

> 💡 **Fun Fact:** Shay Banon built the predecessor "Compass" to help his wife search through her cooking recipes. That hobby project evolved into a company valued at **$11 billion** (Elastic NV).

---

## 🤔 1. Why Elasticsearch? — The Problem It Solves

Traditional databases are **amazing at exact lookups** but **terrible at searching**:

```
┌─────────────────────────────────────────────────────────────┐
│  Traditional Database (SQL)                                  │
│                                                              │
│  SELECT * FROM products WHERE name = 'iPhone 15 Pro Max';   │
│  ✅ Fast! Uses B-Tree index → O(log n) → 2ms               │
│                                                              │
│  SELECT * FROM products WHERE name LIKE '%phone%';          │
│  ❌ Slow! Full Table Scan → O(n) → 45 seconds on 10M rows  │
│                                                              │
│  SELECT * FROM products WHERE name ≈ 'ifone' (typo);        │
│  ❌ Impossible! SQL doesn't understand fuzzy matching        │
│                                                              │
│  "Show me products about 'running shoes' (semantic meaning)" │
│  ❌ Impossible! SQL doesn't understand concepts              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Elasticsearch                                               │
│                                                              │
│  Search: "ifone 15 pro max"                                 │
│  ✅ Found! Fuzzy matching → "iPhone 15 Pro Max" (score: 9.2)│
│                                                              │
│  Search: "running shoes"                                     │
│  ✅ Found! Returns "jogging sneakers", "trail runners" too   │
│     → Understands synonyms, stemming, relevance              │
│                                                              │
│  Search: "error 500 in payment service last 2 hours"        │
│  ✅ Found! Searches 50 billion log lines in < 100ms          │
│                                                              │
│  All responses: < 50ms, even on BILLIONS of documents ⚡    │
└─────────────────────────────────────────────────────────────┘
```

### When to Use Elasticsearch vs a Database

| Scenario | Use ES? | Why |
|----------|---------|-----|
| Search bar on e-commerce site | ✅ **YES** | Fuzzy matching, relevance scoring, autocomplete |
| Centralized log analytics | ✅ **YES** | Search billions of logs in milliseconds |
| Application Performance Monitoring (APM) | ✅ **YES** | Real-time metrics, anomaly detection |
| OLTP transactions (bank transfers) | ❌ **NO** | No ACID transactions, not a primary database |
| Simple key-value lookups | ❌ **NO** | Overkill — use Redis or DynamoDB |
| Complex JOINs across entities | ❌ **NO** | ES is denormalized, not relational |
| Geospatial "find restaurants near me" | ✅ **YES** | Native geo queries with blazing speed |
| Security/SIEM analytics | ✅ **YES** | Real-time threat detection across massive logs |

> ⭐ **Golden Rule:** Elasticsearch is a **search and analytics engine**, NOT a primary database. Use it **alongside** your database, not **instead of** it.

---

## ⚙️ 2. The Magic Behind the Speed — Inverted Index

> **This is the most important concept in search engineering. Understand this, and everything else clicks.**

### 2.1 — Forward Index vs Inverted Index

A traditional database uses a **forward index** — it maps documents to their words:

```
╔══════════════════════════════════════════════════════════╗
║               FORWARD INDEX (Traditional DB)              ║
║                                                           ║
║  Doc 1 → "The quick brown fox jumps over the lazy dog"   ║
║  Doc 2 → "Quick brown dogs are friendly"                 ║
║  Doc 3 → "The fox is quick and clever"                   ║
║                                                           ║
║  🔍 Search "quick fox":                                   ║
║  → Must scan Doc 1... found "quick"✅ found "fox"✅       ║
║  → Must scan Doc 2... found "quick"✅ no "fox"❌          ║
║  → Must scan Doc 3... found "quick"✅ found "fox"✅       ║
║  → Checked ALL documents! 😱 O(n × m)                    ║
╚══════════════════════════════════════════════════════════╝
```

Elasticsearch flips this around with an **Inverted Index** — it maps words to their documents:

```
╔══════════════════════════════════════════════════════════╗
║              INVERTED INDEX (Elasticsearch)               ║
║                                                           ║
║  Term        → Document IDs                               ║
║  ─────────────────────────────                            ║
║  "the"       → [Doc1, Doc3]                               ║
║  "quick"     → [Doc1, Doc2, Doc3]   ← appears in 3 docs  ║
║  "brown"     → [Doc1, Doc2]                               ║
║  "fox"       → [Doc1, Doc3]                               ║
║  "jumps"     → [Doc1]                                     ║
║  "over"      → [Doc1]                                     ║
║  "lazy"      → [Doc1]                                     ║
║  "dog"       → [Doc1]                                     ║
║  "dogs"      → [Doc2]              ← different from "dog" ║
║  "are"       → [Doc2]                                     ║
║  "friendly"  → [Doc2]                                     ║
║  "is"        → [Doc3]                                     ║
║  "and"       → [Doc3]                                     ║
║  "clever"    → [Doc3]                                     ║
║                                                           ║
║  🔍 Search "quick fox":                                   ║
║  → Look up "quick" → [Doc1, Doc2, Doc3]                   ║
║  → Look up "fox"   → [Doc1, Doc3]                         ║
║  → Intersect → [Doc1, Doc3] ✅ DONE!                     ║
║  → Checked ZERO documents! 🚀 O(1) lookup per term       ║
╚══════════════════════════════════════════════════════════╝
```

> 🧠 **Analogy:** A forward index is like reading every page of a textbook to find "photosynthesis." An inverted index is like checking the **index at the back** of the book — it tells you "photosynthesis: pages 45, 112, 203."

### 2.2 — What the Inverted Index Actually Stores

It's not just term → doc IDs. Elasticsearch stores **much more**:

```
┌─────────────────────────────────────────────────────────────┐
│                    INVERTED INDEX (Full)                      │
│                                                              │
│  Term Dictionary (sorted, searchable via FST)                │
│  ┌──────────┬──────────────┬───────────┬──────────────────┐ │
│  │ Term     │ Doc Freq (DF)│ Postings  │ Positions        │ │
│  ├──────────┼──────────────┼───────────┼──────────────────┤ │
│  │ "quick"  │ 3            │ D1,D2,D3  │ D1:[1], D2:[0],  │ │
│  │          │              │           │ D3:[3]           │ │
│  │ "fox"    │ 2            │ D1,D3     │ D1:[3], D3:[1]   │ │
│  │ "brown"  │ 2            │ D1,D2     │ D1:[2], D2:[1]   │ │
│  └──────────┴──────────────┴───────────┴──────────────────┘ │
│                                                              │
│  • Term Dictionary: All unique terms, sorted alphabetically  │
│  • Doc Frequency:   How many docs contain this term          │
│  • Postings List:   Which doc IDs contain this term          │
│  • Positions:       WHERE in the doc the term appears        │
│    (needed for phrase queries like "quick fox")              │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 — How "dog" vs "dogs" Gets Solved (Stemming)

Notice in our raw inverted index, "dog" (Doc1) and "dogs" (Doc2) are **different entries**. If you search "dog", you miss Doc2!

This is where **Analyzers** come in (we'll deep-dive later):

```
Before Analysis:  "Quick Brown Dogs Are Friendly"
                       │
                       ▼
              ┌──────────────────┐
              │   ANALYZER       │
              │                  │
              │  1. Tokenizer    │  → ["Quick","Brown","Dogs","Are","Friendly"]
              │  2. Lowercase    │  → ["quick","brown","dogs","are","friendly"]
              │  3. Stemmer      │  → ["quick","brown","dog","are","friend"]
              │  4. Stop Words   │  → ["quick","brown","dog","friend"]
              │     (remove "are")│
              └──────────────────┘
                       │
                       ▼
After Analysis:  Stored terms → ["quick","brown","dog","friend"]

Now "dog" and "dogs" BOTH map to "dog" → FOUND! ✅
```

---

## 🏗️ 3. Elasticsearch Cluster Architecture

> **Elasticsearch is distributed by nature — it was born to run on clusters.**

### 3.1 — The Building Blocks

```
┌──────────────────────────────────────────────────────────────────┐
│                     ELASTICSEARCH CLUSTER                         │
│                   (Name: "production-search")                     │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   NODE 1     │  │   NODE 2     │  │   NODE 3     │           │
│  │  (Master +   │  │  (Data Node) │  │  (Data Node) │           │
│  │   Data Node) │  │              │  │              │           │
│  │              │  │              │  │              │           │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │           │
│  │ │ Shard P0 │ │  │ │ Shard P1 │ │  │ │ Shard P2 │ │           │
│  │ │ (Primary)│ │  │ │ (Primary)│ │  │ │ (Primary)│ │           │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │           │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │           │
│  │ │ Shard R1 │ │  │ │ Shard R2 │ │  │ │ Shard R0 │ │           │
│  │ │ (Replica)│ │  │ │ (Replica)│ │  │ │ (Replica)│ │           │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                   │
│  Index "products" → 3 Primary Shards + 1 Replica each           │
│  Total: 6 shards distributed across 3 nodes                      │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 — Key Concepts Explained

| Concept | What It Is | Real-World Analogy |
|---------|-----------|-------------------|
| **Cluster** | A group of nodes working together | A team of librarians |
| **Node** | A single Elasticsearch server (JVM instance) | One librarian |
| **Index** | A collection of documents (like a DB table) | One bookshelf topic |
| **Document** | A single JSON record (the unit of data) | One book |
| **Shard** | A slice of an index (a mini Lucene index) | One shelf section |
| **Primary Shard** | The original shard that accepts writes | The original copy |
| **Replica Shard** | A copy of a primary for HA + read throughput | The backup copy |
| **Mapping** | Schema definition for an index | The card catalog format |

### 3.3 — Node Types (Roles)

```
┌────────────────────────────────────────────────────────────────┐
│                    NODE ROLES IN A CLUSTER                      │
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │   MASTER-ELIGIBLE NODE  │  • Manages cluster state          │
│  │   (node.roles: [master])│  • Creates/deletes indices        │
│  │                         │  • Assigns shards to nodes        │
│  │   🧠 The Brain          │  • Tracks which nodes are alive   │
│  └─────────────────────────┘  • Should be lightweight (3-5)    │
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │   DATA NODE             │  • Stores data (shards)           │
│  │   (node.roles: [data])  │  • Executes CRUD, search, agg    │
│  │                         │  • I/O, CPU, memory intensive     │
│  │   💪 The Muscle          │  • Most of your nodes are these   │
│  └─────────────────────────┘                                    │
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │   INGEST NODE           │  • Pre-processes documents        │
│  │   (node.roles: [ingest])│  • Runs pipelines (grok, geoip)  │
│  │                         │  • Transforms before indexing     │
│  │   🏭 The Processor       │  • Like a mini Logstash           │
│  └─────────────────────────┘                                    │
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │   COORDINATING NODE     │  • Routes requests to data nodes  │
│  │   (node.roles: [])      │  • Gathers & merges results       │
│  │                         │  • Acts as a smart load balancer  │
│  │   🗺️ The Router          │  • No data, no master duties      │
│  └─────────────────────────┘                                    │
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │   ML NODE               │  • Runs machine learning jobs     │
│  │   (node.roles: [ml])    │  • Anomaly detection              │
│  │                         │  • Forecasting                    │
│  │   🤖 The Predictor       │  • Requires X-Pack / Platinum     │
│  └─────────────────────────┘                                    │
└────────────────────────────────────────────────────────────────┘
```

> 💡 **Production Tip:** In small clusters (< 10 nodes), one node can wear multiple hats (master + data). In large clusters (50+ nodes), **always separate** master nodes from data nodes. A distracted master = cluster instability.

### 3.4 — Shards — The Heart of Horizontal Scaling

**Why shards exist:** A single server has finite disk, RAM, and CPU. Shards let you split an index across machines.

```
Without Sharding:
┌─────────────────────────────────┐
│  Index: "logs" (5 TB)           │
│  Single Node (16GB RAM, 1TB SSD)│
│  ❌ Won't fit!                   │
│  ❌ Single point of failure      │
│  ❌ One CPU doing all the work   │
└─────────────────────────────────┘

With Sharding (5 Primary Shards):
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Shard 0  │ │ Shard 1  │ │ Shard 2  │ │ Shard 3  │ │ Shard 4  │
│  1 TB    │ │  1 TB    │ │  1 TB    │ │  1 TB    │ │  1 TB    │
│ Node 1   │ │ Node 2   │ │ Node 3   │ │ Node 4   │ │ Node 5   │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
✅ Data spread across 5 nodes
✅ 5x parallel search power
✅ Each node handles 1 TB (manageable)
```

### Shard Routing — How ES Knows Where to Put a Document

```
When you index a document, ES decides which shard gets it:

  shard_number = hash(_routing) % number_of_primary_shards

  Default: _routing = document _id

  Example:
    hash("user_12345") % 5 = 3
    → Document goes to Shard 3

  ⚠️ WARNING: You CANNOT change the number of primary shards
     after index creation (because the hash formula changes)!
     Plan your shard count carefully.
```

### How Many Shards? — The Sizing Guide

| Index Size | Recommended Shards | Why |
|-----------|-------------------|-----|
| < 1 GB | 1 | Overhead of sharding not worth it |
| 1 - 30 GB | 1 - 3 | Sweet spot for most use cases |
| 30 - 200 GB | 3 - 5 | Balance between parallelism and overhead |
| 200 GB - 1 TB | 5 - 10 | Need to spread I/O |
| > 1 TB | 10+ | Each shard ~30-50 GB ideal |

> ⭐ **Golden Rule:** Target **30-50 GB per shard**. Too small (< 1 GB) = overhead waste. Too large (> 50 GB) = slow recovery, slow searches.

### 3.5 — Replicas — Fault Tolerance + Speed

```
Primary Shard 0 (Node 1)          Replica Shard 0 (Node 3)
┌──────────────────────┐          ┌──────────────────────┐
│ ████████████████████ │ ──COPY──▶│ ████████████████████ │
│  The Original Data   │          │  Exact Same Data     │
│                      │          │                      │
│  ✅ Handles writes   │          │  ❌ No writes         │
│  ✅ Handles reads    │          │  ✅ Handles reads     │
└──────────────────────┘          └──────────────────────┘

What happens if Node 1 dies?
┌──────────────────────┐          ┌──────────────────────┐
│ ❌❌❌❌❌ Node 1 DOWN │          │ ████████████████████ │
│   💀 RIP              │          │  🎉 Promoted to      │
│                      │          │  PRIMARY instantly!   │
└──────────────────────┘          └──────────────────────┘
                                  │
                                  ▼ Zero downtime! ✅
```

**Replicas provide TWO benefits:**
1. **High Availability** — If a node dies, replicas take over instantly
2. **Read Throughput** — Search queries can hit replicas in parallel

```json
// Creating an index with shards and replicas
PUT /products
{
  "settings": {
    "number_of_shards": 3,        // 3 primary shards (set at creation, can't change)
    "number_of_replicas": 1       // 1 replica per primary (can change anytime)
  }
}
// Total shards = 3 primaries + 3 replicas = 6 shards
```

> 💡 **Pro Tip:** `number_of_replicas` can be changed dynamically! Start with 1, increase to 2 during peak traffic for more read capacity.

---

## 📄 4. Documents — The Unit of Data

Elasticsearch stores everything as **JSON documents**. No tables. No rows. Just JSON.

```json
// A document in the "products" index
{
  "_index": "products",           // Which index (like a DB table)
  "_id": "prod_001",              // Unique identifier
  "_source": {                    // The actual data (your JSON)
    "name": "MacBook Pro 16-inch",
    "brand": "Apple",
    "price": 2499.99,
    "category": "Laptops",
    "description": "Powerful laptop with M3 Max chip, 36GB RAM, 1TB SSD",
    "tags": ["apple", "laptop", "professional", "m3"],
    "specs": {
      "cpu": "M3 Max",
      "ram_gb": 36,
      "storage_gb": 1024
    },
    "ratings": {
      "average": 4.8,
      "count": 12453
    },
    "in_stock": true,
    "created_at": "2024-01-15T10:30:00Z",
    "location": {
      "lat": 37.3861,
      "lon": -122.0839
    }
  }
}
```

### Document vs Relational Mapping

| Relational (SQL) | Elasticsearch | Notes |
|-------------------|---------------|-------|
| Database | Cluster | One cluster can hold many indices |
| Table | Index | Each index has its own mapping |
| Row | Document | A JSON object |
| Column | Field | A key in the JSON |
| Schema | Mapping | Defines field types and analysis |
| Primary Key | `_id` | Unique per document within an index |

> ⚠️ **Note:** Before ES 7.x, there was an extra concept called **"Type"** (like a sub-table within an index). Types were **removed in ES 7.0+**. One index = one mapping now.

---

## 🗺️ 5. Mappings — The Schema of Elasticsearch

> **"Mappings tell Elasticsearch HOW to interpret your data."**

Elasticsearch can **auto-detect** field types (dynamic mapping), but for production, you should **define mappings explicitly**.

### 5.1 — Core Field Types

| Type | Example | When to Use |
|------|---------|-------------|
| `text` | `"description": "Powerful laptop"` | Full-text search (analyzed, tokenized) |
| `keyword` | `"status": "active"` | Exact match, filtering, sorting, aggregations |
| `integer` / `long` | `"price": 2499` | Numeric ranges, math |
| `float` / `double` | `"rating": 4.8` | Decimal numbers |
| `boolean` | `"in_stock": true` | True/false filtering |
| `date` | `"created_at": "2024-01-15"` | Date ranges, histograms |
| `geo_point` | `{"lat": 37.38, "lon": -122.08}` | Geo queries (distance, bounding box) |
| `nested` | `[{"name": "Ram", "age": 25}]` | Array of objects (preserves relationships) |
| `object` | `{"specs": {"cpu": "M3"}}` | Nested JSON (flattened internally) |
| `ip` | `"client_ip": "192.168.1.1"` | IP address ranges |

### 5.2 — text vs keyword — The #1 Confusion

This trips up EVERYONE. Understanding this difference is critical:

```
┌────────────────────────────────────────────────────────────────┐
│                    text vs keyword                              │
│                                                                 │
│  Field: "name": "MacBook Pro 16-inch"                          │
│                                                                 │
│  ┌─────────────────────────────┐  ┌──────────────────────────┐ │
│  │     TYPE: text              │  │    TYPE: keyword          │ │
│  │                             │  │                           │ │
│  │  Analyzed ✅                │  │  NOT analyzed ❌          │ │
│  │  Tokens: ["macbook",       │  │  Stored as-is:            │ │
│  │           "pro",            │  │  "MacBook Pro 16-inch"   │ │
│  │           "16",             │  │                           │ │
│  │           "inch"]           │  │  Use for:                │ │
│  │                             │  │  • Exact match           │ │
│  │  Use for:                  │  │  • Sorting                │ │
│  │  • Full-text search        │  │  • Aggregations           │ │
│  │  • "Find laptops"          │  │  • Filtering              │ │
│  │  • Relevance scoring       │  │  • "status = active"     │ │
│  └─────────────────────────────┘  └──────────────────────────┘ │
│                                                                 │
│  💡 ES default for strings: Creates BOTH text AND keyword      │
│     as a "multi-field":                                         │
│     "name"     → text (for searching)                          │
│     "name.keyword" → keyword (for sorting/aggs)                │
└────────────────────────────────────────────────────────────────┘
```

### 5.3 — Explicit Mapping Example

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",                    // Full-text searchable
        "analyzer": "standard",
        "fields": {
          "keyword": {                      // Also available as keyword
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "brand": {
        "type": "keyword"                   // Exact match only (no analysis)
      },
      "price": {
        "type": "float"
      },
      "description": {
        "type": "text",
        "analyzer": "english"               // English stemming + stop words
      },
      "tags": {
        "type": "keyword"                   // Array of exact keywords
      },
      "specs": {
        "type": "object",                   // Nested JSON object
        "properties": {
          "cpu": { "type": "keyword" },
          "ram_gb": { "type": "integer" },
          "storage_gb": { "type": "integer" }
        }
      },
      "created_at": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_millis"
      },
      "location": {
        "type": "geo_point"                 // Lat/lon for geo queries
      },
      "in_stock": {
        "type": "boolean"
      }
    }
  }
}
```

> ⚠️ **Critical Warning:** Once a field mapping is created, you **CANNOT change its type**. To change a mapping, you must create a new index and **reindex** your data. Plan your mappings carefully!

### 5.4 — Dynamic Mapping (Auto-Detection)

If you don't define a mapping, ES guesses:

| JSON Value | Auto-Detected Type | Gotcha |
|-----------|-------------------|--------|
| `"hello world"` | `text` + `keyword` (multi-field) | Works fine |
| `42` | `long` | May want `integer` |
| `3.14` | `float` | OK |
| `true` | `boolean` | OK |
| `"2024-01-15"` | `date` | ✅ Detected |
| `"192.168.1.1"` | `text` | ❌ Should be `ip`! |
| `{"lat":37,"lon":-122}` | `object` | ❌ Should be `geo_point`! |

> 💡 **Pro Tip:** Use `dynamic: "strict"` in production to prevent surprise fields:
```json
PUT /products
{
  "mappings": {
    "dynamic": "strict",    // Reject documents with unmapped fields
    "properties": { ... }
  }
}
```

---

## 🔬 6. Analyzers — How Text Becomes Searchable

> **Analyzers are the bridge between human language and the inverted index.**

### 6.1 — The Analysis Pipeline

Every `text` field goes through this pipeline before being stored:

```
         Input: "The Quick Brown Fox-Jumps! Over 2 Lazy Dogs."
                            │
                            ▼
                ┌───────────────────────┐
                │   CHARACTER FILTERS   │  → Strip HTML, replace characters
                │   (Optional, 0+)     │
                └───────────┬───────────┘
                            │  "The Quick Brown Fox-Jumps! Over 2 Lazy Dogs."
                            ▼
                ┌───────────────────────┐
                │      TOKENIZER       │  → Splits text into tokens (words)
                │   (Exactly 1)        │
                └───────────┬───────────┘
                            │  ["The","Quick","Brown","Fox","Jumps","Over","2","Lazy","Dogs"]
                            ▼
                ┌───────────────────────┐
                │    TOKEN FILTERS     │  → Transform tokens (lowercase, stem, etc.)
                │   (Optional, 0+)     │
                └───────────┬───────────┘
                            │  ["the","quick","brown","fox","jump","over","2","lazi","dog"]
                            ▼
                    Stored in Inverted Index
```

### 6.2 — Built-in Analyzers

| Analyzer | Input: "The Quick Brown Foxes!" | Output Tokens |
|----------|-------------------------------|---------------|
| `standard` | Default. Splits on word boundaries, lowercases | `[the, quick, brown, foxes]` |
| `simple` | Splits on non-letters, lowercases | `[the, quick, brown, foxes]` |
| `whitespace` | Splits on whitespace only | `[The, Quick, Brown, Foxes!]` |
| `stop` | Like standard + removes stop words | `[quick, brown, foxes]` |
| `english` | Standard + stemming + stop words | `[quick, brown, fox]` |
| `keyword` | NO analysis — entire string is one token | `[The Quick Brown Foxes!]` |

### 6.3 — Custom Analyzer (Real-World Example)

Building an analyzer for an e-commerce product search:

```json
PUT /products
{
  "settings": {
    "analysis": {
      "char_filter": {
        "remove_special": {
          "type": "pattern_replace",
          "pattern": "[^a-zA-Z0-9\\s]",      // Remove special chars
          "replacement": ""
        }
      },
      "tokenizer": {
        "my_tokenizer": {
          "type": "standard"
        }
      },
      "filter": {
        "my_synonyms": {
          "type": "synonym",
          "synonyms": [
            "laptop,notebook,portable computer",
            "phone,mobile,smartphone,cell phone",
            "tv,television,telly"
          ]
        },
        "my_stemmer": {
          "type": "stemmer",
          "language": "english"
        }
      },
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "char_filter": ["remove_special"],
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "my_synonyms",
            "my_stemmer",
            "trim"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "product_analyzer"          // Use our custom analyzer
      }
    }
  }
}
```

Now searching "notebook" will also find "laptop" results! 🎉

### 6.4 — Testing Analyzers (The `_analyze` API)

You can test any analyzer without indexing data:

```json
// Test the standard analyzer
POST /_analyze
{
  "analyzer": "standard",
  "text": "The Quick Brown Fox-Jumps!"
}
// Result: ["the", "quick", "brown", "fox", "jumps"]

// Test the english analyzer
POST /_analyze
{
  "analyzer": "english",
  "text": "The Quick Brown Foxes are Running"
}
// Result: ["quick", "brown", "fox", "run"]
// Notice: "The" and "are" removed (stop words)
//         "Foxes" → "fox" (stemmed)
//         "Running" → "run" (stemmed)

// Test a specific field's analyzer
POST /products/_analyze
{
  "field": "name",
  "text": "MacBook Pro 16-inch Laptop"
}
```

---

## 📥 7. Indexing Documents — Write Path

> **"Indexing" in ES means adding/updating documents. The write path is carefully designed for durability + speed.**

### 7.1 — The Write Path (Step by Step)

```
Client sends: PUT /products/_doc/1 { "name": "iPhone 15" }
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│  1. COORDINATING NODE receives request                       │
│     → Determines target shard: hash("1") % 3 = Shard 1     │
│     → Forwards to Node holding Primary Shard 1               │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  2. PRIMARY SHARD processes the document                     │
│     a) Validate the document against mapping                 │
│     b) Run through the Analyzer pipeline                     │
│     c) Write to IN-MEMORY BUFFER (fast!)                     │
│     d) Write to TRANSLOG (write-ahead log for crash safety) │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  3. REPLICATE to replica shards                              │
│     → Primary waits for replica acknowledgment               │
│     → Only then responds "success" to client                 │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  4. REFRESH (every 1 second by default)                      │
│     → In-memory buffer → New Lucene SEGMENT (on disk)        │
│     → Document becomes SEARCHABLE ⚡                         │
│     → This is "near real-time" search!                       │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  5. FLUSH (every ~30 minutes or when translog gets large)   │
│     → Lucene COMMIT → segments written to durable storage   │
│     → Translog cleared                                       │
│     → Data is now fully persistent ✅                        │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 — Key Write Concepts

| Concept | What | Default | Impact |
|---------|------|---------|--------|
| **Refresh** | Buffer → Searchable segment | Every **1 second** | Documents are searchable ~1s after indexing |
| **Flush** | Segments committed to disk | ~30 min / translog size | Full durability |
| **Translog** | Write-ahead log | On every write | Crash recovery (like WAL in PostgreSQL) |
| **Merge** | Small segments → bigger segments | Background | Reduces segment count, reclaims deleted space |

> 💡 **Near Real-Time Search:** When you index a document, it's NOT instantly searchable. It becomes searchable after the next **refresh** (~1 second). This is the meaning of "near real-time."

```json
// Force refresh (make all buffered docs searchable NOW)
POST /products/_refresh

// Disable refresh for bulk loading (then re-enable)
PUT /products/_settings
{
  "index.refresh_interval": "-1"    // Disable during bulk load
}
// ... bulk index millions of documents ...
PUT /products/_settings
{
  "index.refresh_interval": "1s"    // Re-enable
}
POST /products/_refresh              // Force refresh
```

---

## 🔍 8. Search Path — How Queries Execute

### 8.1 — The Two Phases: Query Then Fetch

```
Client sends: GET /products/_search { "query": { "match": { "name": "laptop" } } }

═══════════════════════ PHASE 1: QUERY ════════════════════════

Coordinating Node → Broadcasts query to ALL shards (primary or replica)

┌──────────┐     ┌──────────┐     ┌──────────┐
│ Shard 0  │     │ Shard 1  │     │ Shard 2  │
│          │     │          │     │          │
│ Search   │     │ Search   │     │ Search   │
│ local    │     │ local    │     │ local    │
│ inverted │     │ inverted │     │ inverted │
│ index    │     │ index    │     │ index    │
│          │     │          │     │          │
│ Result:  │     │ Result:  │     │ Result:  │
│ DocID: 5 │     │ DocID: 2 │     │ DocID: 8 │
│ Score:9.1│     │ Score:8.7│     │ Score:7.3│
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────────────┼────────────────┘
                      ▼
         Coordinating Node merges results
         Sorted by _score: [DocID5, DocID2, DocID8]
         (Only returns TOP N doc IDs + scores)

═══════════════════════ PHASE 2: FETCH ════════════════════════

Coordinating Node → Fetches FULL documents for TOP N results

GET full document for DocID 5 from Shard 0  → { "name": "MacBook Pro" ... }
GET full document for DocID 2 from Shard 1  → { "name": "Dell XPS 15" ... }
GET full document for DocID 8 from Shard 2  → { "name": "ThinkPad X1" ... }

═══════════════════════ RESPONSE ══════════════════════════════

Return final sorted results to client ✅
```

### 8.2 — Relevance Scoring (BM25)

Elasticsearch uses **BM25** (Best Matching 25) to score how relevant each document is:

```
BM25 Score depends on:

1. TERM FREQUENCY (TF)
   → How often does the search term appear in THIS document?
   → More occurrences = higher score (with diminishing returns)

2. INVERSE DOCUMENT FREQUENCY (IDF)
   → How rare is the search term across ALL documents?
   → Rare terms = higher score ("elasticsearch" > "the")

3. FIELD LENGTH NORMALIZATION
   → Shorter fields score higher for the same term
   → "laptop" in a title (3 words) scores higher than
     "laptop" in a description (200 words)

Example:
  Search: "laptop"
  Doc A: title = "Gaming Laptop"           → Score: 9.2 (short field, exact)
  Doc B: title = "Best Budget Laptops 2024" → Score: 7.8 (longer, stemmed)
  Doc C: description = "...this laptop..."  → Score: 3.1 (long field, buried)
```

---

## 🛠️ 9. CRUD Operations — Hands On

### 9.1 — Index (Create/Update) a Document

```json
// Auto-generated ID
POST /products/_doc
{
  "name": "iPhone 15 Pro",
  "brand": "Apple",
  "price": 1199.99
}

// Specific ID (creates if new, replaces if exists)
PUT /products/_doc/iphone15pro
{
  "name": "iPhone 15 Pro",
  "brand": "Apple",
  "price": 1199.99
}

// Create ONLY (fail if ID already exists)
PUT /products/_create/iphone15pro
{
  "name": "iPhone 15 Pro",
  "brand": "Apple",
  "price": 1199.99
}
```

### 9.2 — Get a Document

```json
// Get by ID
GET /products/_doc/iphone15pro

// Response:
{
  "_index": "products",
  "_id": "iphone15pro",
  "_version": 1,
  "_source": {
    "name": "iPhone 15 Pro",
    "brand": "Apple",
    "price": 1199.99
  }
}

// Get only specific fields
GET /products/_doc/iphone15pro?_source_includes=name,price
```

### 9.3 — Update a Document

```json
// Partial update (merge fields)
POST /products/_update/iphone15pro
{
  "doc": {
    "price": 1099.99,                // Update price
    "in_stock": true                  // Add new field
  }
}

// Script-based update
POST /products/_update/iphone15pro
{
  "script": {
    "source": "ctx._source.price -= params.discount",
    "params": {
      "discount": 100
    }
  }
}
```

### 9.4 — Delete a Document

```json
// Delete by ID
DELETE /products/_doc/iphone15pro

// Delete by query
POST /products/_delete_by_query
{
  "query": {
    "range": {
      "price": { "lt": 10 }          // Delete all products under $10
    }
  }
}
```

### 9.5 — Bulk Operations (Performance King 👑)

```json
// Bulk API — batch multiple operations in ONE request
POST /_bulk
{"index": {"_index": "products", "_id": "1"}}
{"name": "iPhone 15", "brand": "Apple", "price": 999}
{"index": {"_index": "products", "_id": "2"}}
{"name": "Galaxy S24", "brand": "Samsung", "price": 899}
{"update": {"_index": "products", "_id": "1"}}
{"doc": {"price": 949}}
{"delete": {"_index": "products", "_id": "old_product"}}
```

> ⭐ **Performance Tip:** Always use `_bulk` for loading data. Single document indexing: ~500 docs/sec. Bulk indexing: ~10,000-50,000 docs/sec. That's a **100x improvement!**

---

## ⚙️ 10. Index Management

### 10.1 — Index Lifecycle

```json
// Create an index
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s"
  },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "float" }
    }
  }
}

// Check index exists
HEAD /products                // 200 = exists, 404 = doesn't

// Get index settings
GET /products/_settings

// Get index mapping
GET /products/_mapping

// Get index stats
GET /products/_stats

// Close index (saves resources, can't search)
POST /products/_close

// Open index (make searchable again)
POST /products/_open

// Delete index (⚠️ PERMANENT!)
DELETE /products
```

### 10.2 — Index Templates (Auto-Apply Settings)

```json
// Template for all log indices
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],         // Matches logs-2024-01, logs-2024-02, etc.
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "level": { "type": "keyword" },       // INFO, WARN, ERROR
        "message": { "type": "text" },
        "service": { "type": "keyword" },
        "host": { "type": "keyword" },
        "response_time_ms": { "type": "integer" }
      }
    }
  },
  "priority": 100
}

// Now any index matching "logs-*" auto-gets these settings!
POST /logs-2024-06/_doc
{
  "timestamp": "2024-06-15T10:30:00Z",
  "level": "ERROR",
  "message": "Payment gateway timeout after 30s",
  "service": "payment-service",
  "host": "prod-server-05"
}
// 👆 Index "logs-2024-06" auto-created with template settings ✅
```

### 10.3 — Aliases (Zero-Downtime Reindexing)

```json
// Create an alias (virtual name pointing to real index)
POST /_aliases
{
  "actions": [
    { "add": { "index": "products_v1", "alias": "products" } }
  ]
}

// Your app queries "products" (the alias) — doesn't know the real index name!
GET /products/_search { ... }

// Reindex to a new version:
// 1. Create products_v2 with new mappings
// 2. Reindex data: POST /_reindex { "source": {"index":"products_v1"}, "dest": {"index":"products_v2"} }
// 3. Swap the alias (atomic, zero downtime!):
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add":    { "index": "products_v2", "alias": "products" } }
  ]
}
// Your app never notices the swap! ✅
```

---

## 📊 11. Cluster Health & Monitoring

```json
// Quick cluster health
GET /_cluster/health

// Response:
{
  "cluster_name": "production-search",
  "status": "green",              // 🟢 green | 🟡 yellow | 🔴 red
  "number_of_nodes": 5,
  "number_of_data_nodes": 3,
  "active_primary_shards": 15,
  "active_shards": 30,
  "unassigned_shards": 0
}
```

### Cluster Status Explained

```
┌─────────────────────────────────────────────────────────────┐
│  🟢 GREEN   │ All primary AND replica shards are assigned   │
│             │ Everything is perfect. Full redundancy.        │
├─────────────┼───────────────────────────────────────────────┤
│  🟡 YELLOW  │ All primaries assigned, some replicas NOT     │
│             │ Data is complete but not fully redundant.      │
│             │ Common with single-node clusters.              │
│             │ ⚠️ One node failure = potential data loss      │
├─────────────┼───────────────────────────────────────────────┤
│  🔴 RED     │ Some PRIMARY shards are not assigned!         │
│             │ 🚨 EMERGENCY! Data is missing/inaccessible.   │
│             │ Search results may be incomplete.              │
│             │ Fix immediately!                               │
└─────────────┴───────────────────────────────────────────────┘
```

```json
// Node stats
GET /_nodes/stats

// Shard allocation
GET /_cat/shards?v

// Index sizes
GET /_cat/indices?v&s=store.size:desc

// Pending tasks
GET /_cluster/pending_tasks
```

---

## 🔄 12. Elasticsearch vs Other Databases — Positioning

| Feature | Elasticsearch | MongoDB | PostgreSQL | Redis |
|---------|--------------|---------|------------|-------|
| **Primary Use** | Search & Analytics | Document store | Relational OLTP | Caching |
| **Query Speed (search)** | ⚡ Blazing fast | 🟡 Moderate | 🐌 Slow (LIKE%) | N/A |
| **Full-Text Search** | ⭐ Best in class | ✅ Decent (Atlas Search) | ✅ Basic (tsvector) | ❌ |
| **Transactions (ACID)** | ❌ No | ✅ Multi-doc ACID | ✅ Full ACID | ❌ |
| **Aggregations** | ✅ Powerful | ✅ Agg framework | ✅ GROUP BY | ❌ |
| **Geo Queries** | ✅ Native | ✅ GeoJSON | ✅ PostGIS | ✅ Basic |
| **Fuzzy/Typo Tolerance** | ✅ Built-in | ❌ | ❌ | ❌ |
| **Log Analytics** | ⭐ Industry standard | 🟡 Possible | ❌ Not suited | ❌ |
| **Learning Curve** | 🟡 Moderate | 🟢 Easy | 🟡 Moderate | 🟢 Easy |
| **Data Durability** | ✅ (replicas) | ✅ (replica sets) | ✅ (WAL) | 🟡 (optional AOF) |

---

## 🔑 Key Takeaways

```
✅ Elasticsearch is a SEARCH ENGINE built on Apache Lucene — not a primary database
✅ The Inverted Index is the core data structure — maps terms → documents
✅ Analyzers transform text (tokenize, lowercase, stem) before indexing
✅ text fields are for full-text search; keyword fields are for exact match
✅ A Cluster = Nodes → Indices → Shards → Documents
✅ Primary shards hold data; Replica shards provide HA + read scaling
✅ Documents are JSON; Mappings define the schema
✅ Near real-time: documents searchable ~1 second after indexing (refresh)
✅ Always use _bulk API for loading data (100x faster)
✅ Plan your mappings upfront — field types CANNOT be changed after creation
✅ Target 30-50 GB per shard for optimal performance
✅ Green = healthy, Yellow = missing replicas, Red = EMERGENCY
```

---

## ❓ Self-Check Questions

1. Why can't a traditional `LIKE '%keyword%'` query compete with Elasticsearch?
2. What is the difference between a `text` and a `keyword` field?
3. If you have 3 primary shards and 2 replicas, how many total shards exist?
4. What happens during the "refresh" process? Why is ES called "near real-time"?
5. You need to change a field from `text` to `keyword`. What's the process?
6. What does an Analyzer consist of? Name the 3 components.
7. Explain the two-phase search process (Query → Fetch).
8. Why should you never use dynamic mapping in production without `"strict"` mode?
9. What's the purpose of the Translog?
10. A cluster is showing 🟡 YELLOW status — what does this mean and is it urgent?

---

## 🎯 Hands-On Lab

```
Lab 1: Spin up Elasticsearch locally (Docker is easiest)
  → docker run -d --name es -p 9200:9200 -e "discovery.type=single-node" \
       -e "xpack.security.enabled=false" docker.elastic.co/elasticsearch/elasticsearch:8.13.0
  → Verify: curl http://localhost:9200

Lab 2: Create an index "movies" with explicit mapping
  → Fields: title (text), genre (keyword), year (integer),
    rating (float), description (text with english analyzer)

Lab 3: Index 10 movies using the _bulk API

Lab 4: Test the _analyze API
  → Compare "standard" vs "english" analyzer on:
    "The Lord of the Rings: The Fellowship of the Ring"

Lab 5: Check cluster health and explore _cat APIs
  → GET /_cat/indices?v
  → GET /_cat/shards?v
  → GET /_cat/nodes?v
```

---

> **Next Chapter:** [3F.2 — Elasticsearch Querying (Query DSL)](./02-ES-Query-DSL.md) — Master the art of searching: Match, Bool, Range, Aggregations, and more! 🔍
