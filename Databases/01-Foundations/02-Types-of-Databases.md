# 1.2 — Types of Databases — The Big Picture 🟢

> **"Choosing the wrong database is like using a hammer to cut a tree. You'll finish... eventually. And painfully."**

---

## 📌 What You'll Learn

- Every major type of database that exists today
- **When** to use each type (the million-dollar question)
- Real-world examples of who uses what
- How to think about database selection like a senior architect
- The concept of **Polyglot Persistence** — using multiple DBs together

---

## 1. The Two Big Kingdoms: SQL vs NoSQL

Before we dive into specific types, understand the two fundamental philosophies:

```
┌──────────────────────────────────────────────────────────────────┐
│                    THE DATABASE WORLD                              │
│                                                                    │
│         ┌──────────────────┐    ┌──────────────────┐              │
│         │    RELATIONAL     │    │  NON-RELATIONAL   │              │
│         │      (SQL)        │    │     (NoSQL)        │              │
│         ├──────────────────┤    ├──────────────────┤              │
│         │ • Structured      │    │ • Flexible schema  │              │
│         │ • Tables & Rows   │    │ • Various models   │              │
│         │ • Fixed schema    │    │ • Horizontal scale │              │
│         │ • ACID compliant  │    │ • BASE philosophy  │              │
│         │ • SQL language    │    │ • Custom query     │              │
│         │ • Joins           │    │ • Denormalized     │              │
│         │                   │    │                    │              │
│         │ Oracle, MySQL,    │    │ MongoDB, Redis,    │              │
│         │ PostgreSQL,       │    │ Cassandra, Neo4j,  │              │
│         │ SQL Server        │    │ Elasticsearch      │              │
│         └──────────────────┘    └──────────────────┘              │
│                                                                    │
│                    ┌──────────────────┐                            │
│                    │    NewSQL         │                            │
│                    │ (Best of Both)    │                            │
│                    │ CockroachDB,      │                            │
│                    │ TiDB, Spanner     │                            │
│                    └──────────────────┘                            │
└──────────────────────────────────────────────────────────────────┘
```

> ⚠️ **Myth Buster**: "NoSQL" does NOT mean "No SQL." It means **"Not Only SQL."** Many NoSQL databases now support SQL-like queries.

---

## 2. The Complete Database Type Map

Here's every major database type, visualized:

```
                            DATABASES
                               │
            ┌──────────────────┼──────────────────────┐
            │                  │                      │
       RELATIONAL          NoSQL                   SPECIALTY
        (SQL)                │                        │
            │      ┌────────┼────────┐               │
            │      │        │        │        ┌──────┼──────┐
            │  Document  Key-Value  Wide-   Graph  Search  Time-
            │    Store    Store    Column    DB    Engine  Series
            │                      Store
            │
    Oracle, MySQL,   MongoDB  Redis    Cassandra  Neo4j   Elastic  InfluxDB
    PostgreSQL,      CouchDB  DynamoDB HBase      ArangoDB Search  TimescaleDB
    SQL Server,      Firestore Memcached ScyllaDB          Solr
    SQLite, MariaDB
```

---

## 3. Relational Databases (RDBMS) ⭐

### The Concept

Data is organized into **tables** (relations) with **rows** and **columns**. Tables are linked via **relationships** (foreign keys).

### How It Looks

```
TABLE: departments                     TABLE: employees
┌─────┬────────────────┐              ┌─────┬──────────┬────────┬────────┐
│ id  │ name           │              │ id  │ name     │ salary │ dept_id│
├─────┼────────────────┤              ├─────┼──────────┼────────┼────────┤
│  1  │ Engineering    │◄─────────────│ 101 │ Rahul    │ 80000  │   1    │
│  2  │ Marketing      │◄──────┐      │ 102 │ Priya    │ 70000  │   1    │
│  3  │ Finance        │       └──────│ 103 │ John     │ 65000  │   2    │
└─────┴────────────────┘              │ 104 │ Sarah    │ 90000  │   3    │
                                      └─────┴──────────┴────────┴────────┘
                                                              ▲
                                                    Foreign Key (dept_id)
                                                    links to departments.id
```

### Key Characteristics

| Feature | Details |
|---------|---------|
| **Data Model** | Tables with rows and columns |
| **Schema** | Fixed/rigid — define before inserting data |
| **Query Language** | SQL (Structured Query Language) |
| **Relationships** | First-class citizen via JOINs and Foreign Keys |
| **Transactions** | Full ACID support |
| **Scaling** | Primarily vertical (scale up) |
| **Best For** | Structured data with complex relationships |

### Major Players

| Database | Claim to Fame | Used By |
|----------|--------------|---------|
| **Oracle** | Enterprise king, most features | Banks, Telecoms, Fortune 500 |
| **SQL Server** | Microsoft ecosystem integration | Enterprises, .NET shops |
| **MySQL** | Most popular open-source RDBMS | WordPress, Facebook, Uber |
| **PostgreSQL** | Most advanced open-source RDBMS | Apple, Instagram, Spotify |
| **SQLite** | Most deployed DB (embedded) | Every phone, browser, IoT device |
| **MariaDB** | MySQL fork, community-driven | Google, Wikipedia |
| **IBM Db2** | Mainframe legacy + modern | Banks, Airlines |

### ✅ Use When

- Your data is **structured** and relationships matter
- You need **ACID transactions** (banking, healthcare, e-commerce)
- Complex **JOINs** and **aggregations** are frequent
- Data integrity is **non-negotiable**
- Your schema is **relatively stable**

### ❌ Avoid When

- Schema changes every week (rapid prototyping)
- You need to store unstructured data (images, logs, sensor data)
- You need horizontal scaling across 100+ servers
- Your data doesn't have natural relationships

---

## 4. Document Databases 🔥

### The Concept

Data is stored as **documents** (JSON/BSON objects). Each document can have a **different structure**. No rigid schema required.

### How It Looks

```json
// MongoDB Collection: "users"

// Document 1 — A regular user
{
  "_id": "user_001",
  "name": "Rahul Kumar",
  "email": "rahul@example.com",
  "age": 28,
  "address": {
    "city": "Mumbai",
    "state": "Maharashtra",
    "pin": "400001"
  },
  "orders": [
    { "orderId": "ORD-101", "total": 2500, "date": "2026-05-15" },
    { "orderId": "ORD-102", "total": 800,  "date": "2026-06-01" }
  ]
}

// Document 2 — Different fields! No problem.
{
  "_id": "user_002",
  "name": "John Smith",
  "email": "john@example.com",
  "phone": "+1-555-0123",
  "preferences": {
    "theme": "dark",
    "language": "en",
    "notifications": true
  }
}
```

> 💡 Notice how Document 1 has `address` and `orders`, while Document 2 has `phone` and `preferences`. This **flexibility** is the superpower of document databases.

### Key Characteristics

| Feature | Details |
|---------|---------|
| **Data Model** | JSON/BSON documents (nested, hierarchical) |
| **Schema** | Flexible — each document can differ |
| **Query Language** | MongoDB Query Language (MQL), or SQL-like |
| **Relationships** | Embedding (nested docs) or Referencing (like FK) |
| **Transactions** | Multi-document ACID (MongoDB 4.0+) |
| **Scaling** | Horizontal (sharding) — built for it |
| **Best For** | Rapid development, content management, catalogs |

### Major Players

| Database | Specialty |
|----------|----------|
| **MongoDB** | General-purpose, most popular NoSQL DB |
| **CouchDB** | Offline-first, sync between devices |
| **Amazon DocumentDB** | MongoDB-compatible on AWS |
| **Firestore** | Real-time sync, mobile-first (Google) |
| **Couchbase** | Memory-first, mobile edge |

### ✅ Use When

- Schema evolves frequently (startups, MVPs)
- Data is **hierarchical** (user profiles, product catalogs)
- You read **entire objects** at once (not individual fields)
- Need **horizontal scaling** with built-in sharding
- Developer productivity matters (JSON is natural)

### ❌ Avoid When

- Complex JOINs across many entities (use RDBMS)
- Strong relational integrity is needed
- Heavy aggregate reporting (use analytics DBs)

---

## 5. Key-Value Databases ⚡

### The Concept

The simplest model — every piece of data is stored as a **key-value pair**. Think of it as a **giant hash map / dictionary**.

### How It Looks

```
┌──────────────────────────┬──────────────────────────────────────┐
│         KEY               │              VALUE                   │
├──────────────────────────┼──────────────────────────────────────┤
│ user:1001:name           │ "Rahul Kumar"                        │
│ user:1001:session        │ "eyJhbGciOiJIUzI1NiIs..."           │
│ cart:user:1001           │ {"items": [{"id": 5, "qty": 2}]}    │
│ page:home:html           │ "<html>...</html>"                   │
│ rate_limit:192.168.1.1   │ 47                                   │
│ leaderboard:game42       │ [("player1",9500),("player2",9200)] │
└──────────────────────────┴──────────────────────────────────────┘

GET user:1001:name  →  "Rahul Kumar"       (sub-millisecond!)
SET user:1001:name "Rahul K."              (instant!)
DEL user:1001:session                      (logout!)
```

### Key Characteristics

| Feature | Details |
|---------|---------|
| **Data Model** | Key → Value (value can be string, JSON, binary, list, set, etc.) |
| **Schema** | None — value is opaque to the database |
| **Query** | GET/SET/DELETE by key only (no complex queries) |
| **Speed** | Fastest of all DB types (sub-millisecond) |
| **Storage** | Usually in-memory (RAM), optional persistence |
| **Best For** | Caching, sessions, real-time counters, rate limiting |

### Major Players

| Database | Specialty |
|----------|----------|
| **Redis** | Swiss army knife — data structures, pub/sub, streams |
| **Memcached** | Pure caching, ultra-simple, multi-threaded |
| **Amazon DynamoDB** | Serverless key-value + document hybrid |
| **etcd** | Distributed config store (used by Kubernetes) |
| **Riak** | Highly available, distributed |

### ✅ Use When

- You need **sub-millisecond** response times
- Data access is by a **known key** (no complex queries)
- **Caching** database query results
- Storing **sessions**, shopping carts, feature flags
- **Rate limiting**, real-time counters, leaderboards

### ❌ Avoid When

- You need to query by value (search by name, filter by date)
- Complex relationships between data
- You need joins or aggregations
- Data is too large to fit in memory (for in-memory KV stores)

---

## 6. Wide-Column (Column-Family) Databases

### The Concept

Data is stored in **rows** and **dynamic columns**, grouped into **column families**. Think of it as a **2D key-value store** — the row key maps to a set of column-value pairs.

### How It Looks

```
ROW KEY         │ Column Family: "profile"           │ Column Family: "activity"
                │ name      │ email       │ city     │ last_login  │ page_views
────────────────┼───────────┼─────────────┼──────────┼─────────────┼──────────
user:1001       │ Rahul     │ r@mail.com  │ Mumbai   │ 2026-06-02  │ 1547
user:1002       │ Priya     │ p@mail.com  │ -        │ 2026-06-01  │ 892
user:1003       │ John      │ j@mail.com  │ NYC      │ -           │ -

Key difference from RDBMS:
- Each row can have DIFFERENT columns
- Priya has no "city", John has no "last_login" or "page_views"
- Empty cells don't waste storage (sparse data friendly)
```

### Key Characteristics

| Feature | Details |
|---------|---------|
| **Data Model** | Row key → Column families → Columns |
| **Schema** | Semi-structured — column families defined, columns flexible |
| **Query** | By row key and column family (limited queries) |
| **Write Speed** | Extremely fast (append-only, immutable) |
| **Scaling** | Massive horizontal scaling (petabytes, 1000s of nodes) |
| **Best For** | IoT data, time-series, event logging, messaging |

### Major Players

| Database | Specialty |
|----------|----------|
| **Apache Cassandra** | Masterless, multi-datacenter, no single point of failure |
| **Apache HBase** | Hadoop ecosystem, HDFS-backed |
| **ScyllaDB** | Cassandra-compatible, written in C++ (10x faster) |
| **Google Bigtable** | Managed, powers Google Search/Maps/Gmail |

### ✅ Use When

- **Write-heavy** workloads (millions of writes/sec)
- Time-series data (IoT sensors, event logging, metrics)
- Data is **spread across multiple data centers**
- You need **linear horizontal scaling**
- High availability with **zero downtime** is critical

### ❌ Avoid When

- Complex queries, JOINs, aggregations needed
- You need ACID transactions across rows
- Data model changes frequently
- Small dataset (overkill below terabytes)

---

## 7. Graph Databases 🔗

### The Concept

Data is stored as **nodes** (entities) and **edges** (relationships). Relationships are **first-class citizens** — stored directly, not computed via JOINs.

### How It Looks

```
                    ┌─────────────────┐
                    │   Rahul (User)   │
                    │  age: 28         │
                    │  city: Mumbai    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         FOLLOWS         PURCHASED      WORKS_AT
         (since:2024)    (date:2026-05)  (role:Engineer)
              │              │              │
              ▼              ▼              ▼
      ┌──────────────┐ ┌──────────┐ ┌──────────────┐
      │ Priya (User) │ │ iPhone   │ │  TechCorp    │
      │ age: 26      │ │ (Product)│ │  (Company)   │
      └──────────────┘ │ price:   │ │  employees:  │
              │         │ 79999    │ │  5000        │
          REVIEWED      └──────────┘ └──────────────┘
          (stars: 5)
              │
              ▼
        ┌──────────┐
        │ iPhone   │
        │ (Product)│
        └──────────┘

Query: "Find all products bought by friends of Rahul"
→ Rahul —FOLLOWS→ Priya —PURCHASED→ ???
→ This traversal is INSTANT in graph DBs, but requires multiple JOINs in RDBMS
```

### Key Characteristics

| Feature | Details |
|---------|---------|
| **Data Model** | Nodes + Edges (Relationships) + Properties |
| **Schema** | Flexible (property graph model) |
| **Query Language** | Cypher (Neo4j), Gremlin (Apache TinkerPop), SPARQL (RDF) |
| **Relationship Queries** | O(1) per hop — doesn't slow down with data size |
| **Best For** | Social networks, recommendations, fraud detection |

### Major Players

| Database | Specialty |
|----------|----------|
| **Neo4j** | Most popular graph DB, Cypher query language |
| **Amazon Neptune** | Managed, supports Gremlin + SPARQL |
| **ArangoDB** | Multi-model (document + graph + key-value) |
| **TigerGraph** | Enterprise, real-time deep link analytics |
| **JanusGraph** | Open-source, distributed, Gremlin-based |

### ✅ Use When

- **Relationships ARE the data** (social graphs, org charts)
- Recommendation engines ("people who bought X also bought Y")
- **Fraud detection** (find suspicious transaction chains)
- Knowledge graphs, network topology, dependency analysis
- You need **recursive/path queries** that would need 10+ JOINs in SQL

### ❌ Avoid When

- Simple CRUD with few relationships
- Bulk analytics/aggregations
- The data has no meaningful connections
- Simple key-value lookups

---

## 8. Search Engines (Full-Text Search Databases) 🔍

### The Concept

Optimized for **full-text search** and **analytics**. Data is stored in an **inverted index** — like the index at the back of a book but for every word in every document.

### How It Looks

```
Document 1: "MongoDB is a popular NoSQL database"
Document 2: "PostgreSQL is a powerful relational database"
Document 3: "Redis is a popular in-memory key-value store"

INVERTED INDEX:
┌──────────────┬─────────────────────┐
│    WORD       │  DOCUMENT IDs       │
├──────────────┼─────────────────────┤
│ "popular"    │ Doc 1, Doc 3        │
│ "database"   │ Doc 1, Doc 2        │
│ "powerful"   │ Doc 2               │
│ "NoSQL"      │ Doc 1               │
│ "relational" │ Doc 2               │
│ "redis"      │ Doc 3               │
│ "mongodb"    │ Doc 1               │
│ "postgresql" │ Doc 2               │
│ "key-value"  │ Doc 3               │
│ "memory"     │ Doc 3               │
└──────────────┴─────────────────────┘

Search "popular database" → Doc 1 (matches both!) ranked first
                          → Doc 2 (matches "database")
                          → Doc 3 (matches "popular")
```

### Major Players

| Database | Specialty |
|----------|----------|
| **Elasticsearch** | Most popular, part of ELK stack, real-time analytics |
| **Apache Solr** | Battle-tested, Lucene-based, enterprise search |
| **Meilisearch** | Lightning fast, developer-friendly, typo-tolerant |
| **Typesense** | Open-source alternative to Algolia |
| **Algolia** | Hosted search-as-a-service (SaaS) |

### ✅ Use When

- Full-text search across millions of documents
- Log analysis and monitoring (ELK stack)
- Auto-complete, fuzzy matching, faceted search
- Real-time analytics dashboards

### ❌ Avoid When

- Primary datastore (use alongside another DB)
- ACID transactions needed
- Simple lookups by ID (use key-value)
- Frequent updates to the same documents

---

## 9. Time-Series Databases ⏰

### The Concept

Optimized for **time-stamped data** — metrics, events, sensor readings. Data comes in **append-only** streams and queries are almost always **time-range based**.

### How It Looks

```
MEASUREMENT: cpu_usage
┌─────────────────────┬──────────┬──────────┬──────────┐
│      TIMESTAMP       │ server   │  region  │  value   │
├─────────────────────┼──────────┼──────────┼──────────┤
│ 2026-06-02 10:00:00 │ web-01   │ us-east  │  72.5    │
│ 2026-06-02 10:00:01 │ web-01   │ us-east  │  73.1    │
│ 2026-06-02 10:00:02 │ web-01   │ us-east  │  71.8    │
│ 2026-06-02 10:00:00 │ web-02   │ eu-west  │  45.2    │
│ 2026-06-02 10:00:01 │ web-02   │ eu-west  │  46.0    │
└─────────────────────┴──────────┴──────────┴──────────┘

Typical Queries:
→ "Average CPU over last 5 minutes per server"
→ "Max memory usage today in us-east region"
→ "Alert if CPU > 90% for 3 consecutive minutes"
```

### Major Players

| Database | Specialty |
|----------|----------|
| **InfluxDB** | Purpose-built, Flux query language |
| **TimescaleDB** | PostgreSQL extension (full SQL!) |
| **Prometheus** | Monitoring & alerting (pull-based) |
| **QuestDB** | Extremely fast ingestion, SQL-native |
| **Amazon Timestream** | Managed, serverless time-series |

### ✅ Use When

- IoT sensor data, application metrics, server monitoring
- Financial tick data, stock prices
- DevOps monitoring (Prometheus, Grafana stack)
- Data that's append-mostly and queried by time ranges

### ❌ Avoid When

- Data doesn't have a time component
- Complex relationships between entities
- Random updates to old records

---

## 10. Vector Databases 🤖 (The AI Era)

### The Concept

Store and search **high-dimensional vectors** (embeddings). Used in AI/ML for **semantic similarity search** — "find things that are similar in meaning, not just matching keywords."

### How It Looks

```
Traditional DB:  "Find documents containing the word 'happy'"
                 → Only matches exact word "happy"

Vector DB:       "Find documents SIMILAR TO 'happy'"
                 → Returns: "joyful", "delighted", "cheerful", "excited"
                 → Because their vector embeddings are CLOSE in space

                     happy  [0.82, 0.15, 0.93, 0.44, ...]   ─┐
                     joyful [0.80, 0.17, 0.91, 0.42, ...]    │  CLOSE!
                     sad    [0.12, 0.88, 0.05, 0.67, ...]    │  FAR!
                     angry  [0.15, 0.91, 0.08, 0.71, ...]   ─┘  FAR!
```

### Major Players

| Database | Specialty |
|----------|----------|
| **Pinecone** | Fully managed, serverless, production-ready |
| **Milvus** | Open-source, high-performance |
| **Weaviate** | Open-source, built-in vectorization |
| **Qdrant** | Rust-based, high performance |
| **pgvector** | PostgreSQL extension (add vectors to existing PG!) |
| **Chroma** | Lightweight, developer-friendly |

### ✅ Use When

- AI-powered semantic search
- RAG (Retrieval-Augmented Generation) for LLMs
- Image/video/audio similarity search
- Recommendation systems based on content similarity
- Anomaly detection in high-dimensional data

### ❌ Avoid When

- You don't have embedding vectors
- Simple keyword search is sufficient
- Exact matching is needed (use traditional DB)

---

## 11. NewSQL Databases 🆕

### The Concept

**SQL interface + NoSQL scalability.** Get full ACID transactions, JOINs, and SQL syntax — but with horizontal scaling across hundreds of nodes worldwide.

### The Gap NewSQL Fills

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   SQL:     ✅ ACID  ✅ SQL  ✅ JOINs   ❌ Horizontal Scale  │
│   NoSQL:   ❌ ACID  ❌ SQL  ❌ JOINs   ✅ Horizontal Scale  │
│   NewSQL:  ✅ ACID  ✅ SQL  ✅ JOINs   ✅ Horizontal Scale  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Major Players

| Database | Specialty |
|----------|----------|
| **CockroachDB** | Survive any failure, PostgreSQL-compatible |
| **TiDB** | MySQL-compatible, HTAP (analytics + transactions) |
| **Google Spanner** | Global consistency with TrueTime, managed |
| **YugabyteDB** | PG + Cassandra compatible, open-source |
| **VoltDB** | In-memory, extreme speed |

### ✅ Use When

- Need SQL + horizontal scaling (the best of both worlds)
- **Global applications** needing multi-region data
- Outgrowing single-node PostgreSQL/MySQL
- Need **strong consistency** across distributed nodes

### ❌ Avoid When

- Single-server workload (overkill — just use PostgreSQL)
- Team doesn't need distributed features
- Cost is a concern (distributed = more infrastructure)

---

## 12. The Complete Comparison Matrix

### By Data Model

| Type | Shape of Data | Example | Query Pattern |
|------|--------------|---------|---------------|
| Relational | Tables (rows × columns) | Oracle, PostgreSQL | SQL (any complex query) |
| Document | JSON documents | MongoDB | By field, nested queries |
| Key-Value | Key → Value pairs | Redis | GET/SET by key |
| Wide-Column | Row key → Column families | Cassandra | By partition key |
| Graph | Nodes + Edges | Neo4j | Traversal (path/pattern) |
| Search | Inverted index | Elasticsearch | Full-text search |
| Time-Series | Timestamped measurements | InfluxDB | Time-range queries |
| Vector | High-dimensional vectors | Pinecone | Similarity (ANN) search |

### By Use Case (Decision Matrix)

```
What's your PRIMARY need?
│
├── Structured data + Complex relationships?
│   └── ✅ RELATIONAL (PostgreSQL, MySQL, Oracle, SQL Server)
│
├── Flexible schema + Rapid development?
│   └── ✅ DOCUMENT (MongoDB, Firestore)
│
├── Blazing fast reads/writes by key?
│   └── ✅ KEY-VALUE (Redis, DynamoDB)
│
├── Massive write throughput + Multi-datacenter?
│   └── ✅ WIDE-COLUMN (Cassandra, ScyllaDB)
│
├── Deep relationship traversal?
│   └── ✅ GRAPH (Neo4j, Neptune)
│
├── Full-text search + Log analytics?
│   └── ✅ SEARCH ENGINE (Elasticsearch)
│
├── Metrics, IoT, Time-stamped data?
│   └── ✅ TIME-SERIES (InfluxDB, TimescaleDB)
│
├── AI/ML similarity search?
│   └── ✅ VECTOR (Pinecone, pgvector)
│
└── SQL + Horizontal scaling globally?
    └── ✅ NEWSQL (CockroachDB, TiDB, Spanner)
```

---

## 13. Polyglot Persistence — Real World Uses Multiple Databases

No single database does everything well. Modern systems use **multiple databases** together:

### Example: E-Commerce Platform Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE PLATFORM                             │
│                                                                    │
│  ┌─────────────┐  Users, Orders, Payments                         │
│  │ PostgreSQL   │  (Strong ACID, complex JOINs)                   │
│  └─────────────┘                                                  │
│                                                                    │
│  ┌─────────────┐  Product Catalog (flexible attributes)           │
│  │  MongoDB     │  (Each product category has different fields)   │
│  └─────────────┘                                                  │
│                                                                    │
│  ┌─────────────┐  Session cache, Shopping cart, Rate limiting     │
│  │   Redis      │  (Sub-millisecond speed, TTL support)          │
│  └─────────────┘                                                  │
│                                                                    │
│  ┌──────────────┐  Product search, Auto-complete                  │
│  │Elasticsearch │  (Full-text, fuzzy, faceted search)             │
│  └──────────────┘                                                  │
│                                                                    │
│  ┌─────────────┐  "Customers who bought X also bought Y"         │
│  │   Neo4j      │  (Recommendation engine via graph traversal)   │
│  └─────────────┘                                                  │
│                                                                    │
│  ┌─────────────┐  Click-stream analytics, real-time metrics      │
│  │ InfluxDB     │  (Time-series: page views, conversions/sec)    │
│  └─────────────┘                                                  │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

> 💡 **Pro Tip**: Don't start with 6 databases. Start with **one** (usually PostgreSQL or MongoDB), then add specialized databases as specific needs arise.

---

## 14. Database Popularity & Industry Trends (2025–2026)

### DB-Engines Ranking (Approximate)

| Rank | Database | Type | Trend |
|------|----------|------|-------|
| 1 | Oracle | Relational | Stable |
| 2 | MySQL | Relational | Stable |
| 3 | SQL Server | Relational | Stable |
| 4 | PostgreSQL | Relational | 📈 Rising Fast |
| 5 | MongoDB | Document | 📈 Rising |
| 6 | Redis | Key-Value | Stable |
| 7 | Elasticsearch | Search | Stable |
| 8 | SQLite | Relational | 📈 Rising |
| 9 | Cassandra | Wide-Column | Stable |
| 10 | DynamoDB | Key-Value/Doc | 📈 Rising |
| — | PostgreSQL + pgvector | Vector | 🚀 Exploding |
| — | CockroachDB | NewSQL | 📈 Rising |

### Key Trends to Watch

```
🔥 PostgreSQL is eating the database world
   → Best RDBMS + JSONB + pgvector + PostGIS + TimescaleDB
   → One database, many use cases

🔥 Vector databases are the hottest category (AI/LLM era)

🔥 Serverless databases (DynamoDB, PlanetScale, Neon, Turso)
   → Pay per query, zero administration

🔥 Data lakehouse replacing traditional data warehouses
   → Delta Lake, Apache Iceberg

🔥 Edge databases (SQLite on the edge, Turso, Cloudflare D1)
   → Data close to users = low latency
```

---

## 🧠 Quick Recall — Chapter Summary

| Database Type | Data Model | Best For | Example |
|---------------|-----------|----------|---------|
| Relational | Tables | Structured data + relationships | PostgreSQL |
| Document | JSON docs | Flexible schema, nested data | MongoDB |
| Key-Value | Key → Value | Caching, sessions, counters | Redis |
| Wide-Column | Column families | Write-heavy, time-series | Cassandra |
| Graph | Nodes + Edges | Relationships, recommendations | Neo4j |
| Search Engine | Inverted index | Full-text search, logs | Elasticsearch |
| Time-Series | Timestamped rows | IoT, metrics, monitoring | InfluxDB |
| Vector | Embeddings | AI similarity search | Pinecone |
| NewSQL | Tables (distributed) | SQL + horizontal scale | CockroachDB |

---

## ❓ Self-Check Questions

1. When would you choose MongoDB over PostgreSQL?
2. Why can't Redis replace a relational database?
3. What is polyglot persistence? Give an example.
4. When is a graph database better than an RDBMS?
5. What gap does NewSQL fill?
6. Name 3 scenarios where a time-series database is the best fit.
7. What's the difference between a document DB and a wide-column store?
8. Why are vector databases suddenly so popular?

---

> **Next Chapter** → [1.3 — DBMS Architecture & How Databases Work Internally](./03-DBMS-Architecture.md)
