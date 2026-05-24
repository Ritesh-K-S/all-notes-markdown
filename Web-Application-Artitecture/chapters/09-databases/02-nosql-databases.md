# NoSQL Databases — Types & When to Use What

> **What you'll learn**: Why relational databases aren't always the answer, the four main types of NoSQL databases, how they differ from SQL databases, and a decision framework for choosing the right one.

---

## Real-Life Analogy

Imagine you run a **giant warehouse** with different storage needs:

- **Filing cabinets** (Relational DB) — Perfect for tax documents, medical records. Everything has a fixed format, clearly labeled folders, cross-references between files. But what if you need to store things that don't fit into neat folders?

- **Shoe boxes** (Document Store) — Throw anything in: receipts, photos, letters. Each box can hold completely different stuff. Great for flexible, varied content.

- **Mailboxes** (Key-Value Store) — You know the address (key), you get the mail (value). Super fast, no browsing needed.

- **Giant spreadsheets** (Column Store) — Perfect when you need to read one column across millions of rows ("What's the average temperature for the last 10 years?").

- **Corkboard with strings** (Graph DB) — Connecting people, places, and events with lines. "Who knows who? How are these related?"

NoSQL databases are like having **specialized storage systems** for different types of data, instead of forcing everything into filing cabinets.

---

## Core Concept Explained Step-by-Step

### Why NoSQL Exists

Relational databases were designed in the 1970s when:
- Data was structured and predictable
- Applications ran on a single powerful server
- Consistency was more important than speed
- A few thousand users was "a lot"

But modern applications need:
- **Flexible schemas** (user profiles with varying fields)
- **Massive scale** (millions of writes per second)
- **Low latency** (sub-millisecond responses)
- **Global distribution** (data near users worldwide)
- **Availability over consistency** (better to show slightly stale data than nothing)

### The Four Types of NoSQL Databases

```
┌─────────────────────────────────────────────────────────────────┐
│                     NoSQL DATABASE TYPES                          │
├────────────────┬────────────────┬───────────────┬───────────────┤
│   DOCUMENT     │   KEY-VALUE    │ COLUMN-FAMILY │    GRAPH      │
│   STORES       │   STORES       │   STORES      │  DATABASES    │
├────────────────┼────────────────┼───────────────┼───────────────┤
│ MongoDB        │ Redis          │ Cassandra     │ Neo4j         │
│ CouchDB        │ DynamoDB       │ HBase         │ Neptune       │
│ Firestore      │ Memcached      │ ScyllaDB      │ ArangoDB      │
├────────────────┼────────────────┼───────────────┼───────────────┤
│ JSON-like docs │ Simple get/set │ Wide columns  │ Nodes & edges │
│ Flexible schema│ Blazing fast   │ Write-heavy   │ Relationships │
│ Query by field │ No queries     │ Time-series   │ Traversals    │
└────────────────┴────────────────┴───────────────┴───────────────┘
```

### How NoSQL Differs from SQL

```
SQL (Relational)                         NoSQL
┌────────────────────┐                  ┌────────────────────────┐
│ Fixed Schema       │                  │ Flexible/Dynamic Schema│
│ (Define first,     │                  │ (Store first, define   │
│  insert later)     │                  │  later if needed)      │
├────────────────────┤                  ├────────────────────────┤
│ Vertical Scaling   │                  │ Horizontal Scaling     │
│ (Bigger server)    │                  │ (More servers)         │
├────────────────────┤                  ├────────────────────────┤
│ ACID Transactions  │                  │ BASE (Eventually       │
│ (Strong consistency)│                 │  Consistent)           │
├────────────────────┤                  ├────────────────────────┤
│ JOINs across tables│                  │ Denormalized           │
│ (Normalized data)  │                  │ (Embedded/duplicated)  │
├────────────────────┤                  ├────────────────────────┤
│ SQL Language       │                  │ Various APIs           │
│ (Standardized)     │                  │ (DB-specific)          │
└────────────────────┘                  └────────────────────────┘
```

### The CAP Theorem Connection

Every distributed database can only guarantee 2 of these 3:

```
                    CONSISTENCY
                        ▲
                       / \
                      /   \
                     /     \
                    / CA    \  CP
                   / MySQL   \ MongoDB
                  / PostgreSQL \ HBase
                 /─────────────────\
                /                    \
               /         AP          \
              /   Cassandra, DynamoDB \
             /     CouchDB, Riak      \
            ▼─────────────────────────▼
     AVAILABILITY               PARTITION
                                TOLERANCE
```

> **Note**: In practice, network partitions WILL happen, so you're really choosing between **CP** (consistent but might be unavailable) or **AP** (available but might serve stale data).

---

## How It Works Internally

### Data Distribution — Sharding by Default

Unlike SQL databases that typically run on one server, NoSQL databases are designed to spread data across multiple nodes:

```
┌─────────────────────────────────────────────┐
│            Application Layer                  │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│            Router / Driver                    │
│   (Knows which shard has which data)        │
└───┬──────────┬──────────┬──────────┬────────┘
    │          │          │          │
    ▼          ▼          ▼          ▼
┌────────┐┌────────┐┌────────┐┌────────┐
│Shard 1 ││Shard 2 ││Shard 3 ││Shard 4 │
│Users   ││Users   ││Users   ││Users   │
│A-F     ││G-M     ││N-S     ││T-Z     │
└────────┘└────────┘└────────┘└────────┘
```

### Consistency Models

```
STRONG CONSISTENCY:          EVENTUAL CONSISTENCY:
Write ──▶ All nodes         Write ──▶ One node ──▶ Eventually all nodes
          acknowledge                  acknowledges
          before response              immediately

Client sees: Latest data    Client sees: Maybe stale data
Latency: High               Latency: Low
Available: Maybe not        Available: Always

Example: Bank balance       Example: Social media likes count
```

### Storage Approaches Comparison

| Type | Storage Structure | Best For | Worst For |
|------|------------------|----------|-----------|
| Document | B-Tree / LSM-Tree with JSON | Flexible entities | Complex joins |
| Key-Value | Hash table / LSM-Tree | Cache, sessions | Range queries |
| Column-Family | LSM-Tree / SSTables | Analytics, time-series | Random reads |
| Graph | Adjacency lists + index | Relationships | Bulk data |

---

## Code Examples

### Python — Choosing the Right NoSQL Database

```python
# DOCUMENT STORE (MongoDB) — Flexible user profiles
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["myapp"]

# Each user document can have different fields!
db.users.insert_one({
    "name": "Alice",
    "email": "alice@mail.co",
    "preferences": {"theme": "dark", "language": "en"},
    "social_links": ["twitter.com/alice", "github.com/alice"]
})

# KEY-VALUE STORE (Redis) — Session storage
import redis
r = redis.Redis(host='localhost', port=6379)

# Simple: key → value. Blazing fast.
r.set("session:abc123", '{"user_id": 1, "role": "admin"}')
r.expire("session:abc123", 3600)  # Expires in 1 hour
session = r.get("session:abc123")

# COLUMN-FAMILY (Cassandra) — Time-series sensor data
from cassandra.cluster import Cluster

cluster = Cluster(['localhost'])
session = cluster.connect('iot_data')

# Optimized for writes: millions of sensor readings per second
session.execute("""
    INSERT INTO sensor_readings (sensor_id, timestamp, temperature, humidity)
    VALUES (%s, %s, %s, %s)
""", ("sensor-42", "2024-01-15T10:30:00", 22.5, 65.0))

# GRAPH DATABASE (Neo4j) — Social network
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))
with driver.session() as session:
    # Find friends of friends (impossible efficiently in SQL!)
    result = session.run("""
        MATCH (me:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend)-[:FRIENDS_WITH]->(fof)
        WHERE fof <> me AND NOT (me)-[:FRIENDS_WITH]->(fof)
        RETURN fof.name AS suggestion
    """)
    for record in result:
        print(f"Suggested friend: {record['suggestion']}")
```

### Java — Document Store with MongoDB

```java
import com.mongodb.client.*;
import org.bson.Document;
import java.util.Arrays;

public class NoSQLExample {
    public static void main(String[] args) {
        // MongoDB — flexible documents
        MongoClient client = MongoClients.create("mongodb://localhost:27017");
        MongoDatabase db = client.getDatabase("myapp");
        MongoCollection<Document> users = db.getCollection("users");

        // Insert document with nested objects and arrays
        Document user = new Document("name", "Bob")
            .append("email", "bob@mail.co")
            .append("preferences", new Document("theme", "light"))
            .append("tags", Arrays.asList("developer", "gamer"));
        
        users.insertOne(user);

        // Query with filters
        Document found = users.find(new Document("name", "Bob")).first();
        System.out.println("Found: " + found.toJson());

        // Update nested field
        users.updateOne(
            new Document("name", "Bob"),
            new Document("$set", new Document("preferences.theme", "dark"))
        );
    }
}
```

---

## The Decision Framework

```
START HERE: What does your data look like?
│
├── Structured with relationships? ──▶ SQL (PostgreSQL/MySQL)
│
├── Flexible JSON-like documents? ──▶ Document Store (MongoDB)
│
├── Simple key → value lookups? ──▶ Key-Value (Redis/DynamoDB)
│
├── Time-series / Write-heavy? ──▶ Column-Family (Cassandra)
│
├── Complex relationships / traversals? ──▶ Graph (Neo4j)
│
└── Full-text search? ──▶ Search Engine (Elasticsearch)
```

### Detailed Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHEN TO USE WHICH DATABASE?                            │
├──────────────────┬──────────────────────────────────────────────────────┤
│ USE CASE         │ BEST CHOICE              │ WHY                       │
├──────────────────┼──────────────────────────┼───────────────────────────┤
│ E-commerce       │ PostgreSQL + Redis       │ Transactions + caching    │
│ Social network   │ MongoDB + Neo4j + Redis  │ Flex profiles + graphs    │
│ IoT / Sensors    │ Cassandra / TimescaleDB  │ Massive write throughput  │
│ Gaming leaderboard│ Redis                   │ Sorted sets, sub-ms       │
│ Content mgmt     │ MongoDB                  │ Flexible content types    │
│ Chat / Messaging │ Cassandra + Redis        │ Time-ordered messages     │
│ Fraud detection  │ Neo4j + PostgreSQL       │ Pattern matching          │
│ Logging          │ Elasticsearch            │ Full-text search          │
│ User sessions    │ Redis / DynamoDB         │ Fast TTL-based expiry     │
│ Financial        │ PostgreSQL / CockroachDB │ ACID is non-negotiable    │
└──────────────────┴──────────────────────────┴───────────────────────────┘
```

---

## Real-World Example

### Netflix — Polyglot Persistence at Scale

Netflix uses MULTIPLE databases, each for what it does best:

```
┌─────────────────────────────────────────────────────────────┐
│                   Netflix Data Layer                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  User profiles, viewing history            │
│  │ Cassandra   │  → 500+ billion rows, 30PB of data        │
│  │             │  → Chosen for: write throughput, global     │
│  └─────────────┘                                            │
│                                                              │
│  ┌─────────────┐  Caching: session, catalog metadata        │
│  │ Redis/      │  → Millions of ops/second                  │
│  │ Memcached   │  → Chosen for: sub-ms latency             │
│  └─────────────┘                                            │
│                                                              │
│  ┌─────────────┐  Billing, subscriptions, contracts         │
│  │ MySQL       │  → ACID transactions required              │
│  │             │  → Chosen for: consistency, joins          │
│  └─────────────┘                                            │
│                                                              │
│  ┌─────────────┐  Search (movies, actors, genres)           │
│  │Elasticsearch│  → Full-text search with relevance         │
│  │             │  → Chosen for: search speed, fuzzy match   │
│  └─────────────┘                                            │
│                                                              │
│  ┌─────────────┐  Event streaming (views, clicks)           │
│  │ Kafka       │  → Trillions of messages/day               │
│  │             │  → Chosen for: event-driven architecture   │
│  └─────────────┘                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Better Approach |
|---------|-------------|-----------------|
| "NoSQL means no SQL" | NoSQL = "Not Only SQL". Many support SQL-like queries | Understand it means flexible data models |
| Using NoSQL for everything | Transactions, JOINs, and constraints matter | Use SQL for transactional data |
| Ignoring data modeling | "Schemaless" doesn't mean "no thought needed" | Design access patterns BEFORE choosing DB |
| Not planning for consistency | "Eventually consistent" can cause real bugs | Know your consistency requirements |
| Choosing based on hype | "Everyone uses MongoDB" isn't a valid reason | Choose based on YOUR data access patterns |
| Not considering operations | Some NoSQL DBs are hard to manage at scale | Factor in operational complexity |
| Premature NoSQL adoption | SQL handles 90% of apps perfectly | Start with PostgreSQL, add NoSQL when needed |

---

## When to Use / When NOT to Use

### ✅ Use NoSQL When:
- Data schema varies significantly across records
- You need horizontal scaling across many servers
- Write throughput requirements exceed what SQL can handle
- Availability matters more than immediate consistency
- Access patterns are simple (key lookups, time-range queries)
- Data is naturally non-relational (documents, graphs, time-series)

### ❌ Do NOT Use NoSQL When:
- You need complex JOINs across multiple entities
- ACID transactions are critical (banking, inventory)
- Data integrity constraints are important
- Your team is familiar with SQL and the data fits tables
- Your data fits comfortably on one server (under ~1TB)
- You need ad-hoc analytical queries

---

## Key Takeaways

1. **NoSQL is not a replacement for SQL** — it's an alternative for specific use cases where relational databases struggle.
2. **Four main types**: Document (flexible data), Key-Value (speed), Column-Family (write-heavy), Graph (relationships).
3. **Choose based on access patterns** — how you READ data matters more than how you WRITE it.
4. **The CAP theorem** means distributed NoSQL databases trade off between consistency and availability.
5. **Most real systems use MULTIPLE databases** (polyglot persistence) — SQL AND NoSQL together.
6. **Start with PostgreSQL** — only add NoSQL when you have a specific problem that SQL can't solve well.
7. **"Schemaless" is a myth** — your application code still enforces a schema; it's just not in the database.

---

## What's Next?

Next, we'll dive deep into **Document Stores (MongoDB, CouchDB)** (Chapter 9.3), where you'll learn how document databases work internally, when they shine, and how to model data effectively for document-oriented storage.
