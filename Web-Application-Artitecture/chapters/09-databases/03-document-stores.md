# Document Stores (MongoDB, CouchDB)

> **What you'll learn**: How document databases store data as flexible JSON-like documents, when they outperform relational databases, how to model data for document stores, and the internals of MongoDB's storage engine.

---

## Real-Life Analogy

Imagine a **folder system** where each folder contains a **complete dossier** about one thing:

- **Folder: "Alice Johnson"** contains her resume, photo, reference letters, certifications — all different formats, all in one place.
- **Folder: "Bob's Pizza Order"** contains the order details, delivery address, payment info, special instructions — everything needed to fulfill that order.

You don't need to look in 5 different filing cabinets to get all the information. It's all **RIGHT THERE in one folder**.

Compare this to a relational database where Alice's data would be split across the `users` table, `addresses` table, `certifications` table, `references` table... needing 4 JOINs to get her full picture.

A **document store** keeps everything about one entity together in one "document" — just like those complete folders.

---

## Core Concept Explained Step-by-Step

### What is a Document?

A **document** is a self-contained piece of data, usually in JSON (or BSON) format:

```json
{
  "_id": "user_12345",
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "age": 28,
  "address": {
    "street": "123 Main St",
    "city": "Mumbai",
    "zip": "400001"
  },
  "orders": [
    {"item": "Laptop", "price": 999, "date": "2024-01-15"},
    {"item": "Mouse", "price": 29, "date": "2024-01-20"}
  ],
  "tags": ["developer", "premium_member"],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}
```

Notice: **Nested objects**, **arrays**, **varied data types** — all in one document!

### Documents vs Rows

```
RELATIONAL (3 tables, JOINs needed):         DOCUMENT (1 document, everything together):
                                              
┌──────────────────┐                          ┌─────────────────────────────────┐
│ users            │                          │ {                               │
│ id | name | age  │                          │   "name": "Alice",              │
│ 1  | Alice| 28   │──┐                      │   "age": 28,                    │
└──────────────────┘  │                       │   "address": {                  │
                      │                       │     "city": "Mumbai"            │
┌──────────────────┐  │                       │   },                            │
│ addresses        │  │                       │   "orders": [                   │
│ user_id | city   │◀─┘                      │     {"item": "Laptop", ...},    │
│ 1       | Mumbai │──┐                      │     {"item": "Mouse", ...}      │
└──────────────────┘  │                       │   ]                             │
                      │                       │ }                               │
┌──────────────────┐  │                       └─────────────────────────────────┘
│ orders           │  │                       
│ user_id | item   │◀─┘                      One read = ALL the data!
│ 1       | Laptop │                          No JOINs needed!
│ 1       | Mouse  │
└──────────────────┘

Three reads + JOIN = All the data
```

### Collections (Not Tables)

Documents are grouped into **collections** (similar to tables but without enforced schema):

```
DATABASE: "ecommerce"
│
├── COLLECTION: "users"
│   ├── Document: { name: "Alice", age: 28, ... }
│   ├── Document: { name: "Bob", age: 35, premium: true, ... }  ← Has extra field!
│   └── Document: { name: "Charlie", company: "Acme", ... }     ← Different structure!
│
├── COLLECTION: "products"
│   ├── Document: { name: "Laptop", specs: {...}, variants: [...] }
│   └── Document: { name: "T-Shirt", sizes: ["S","M","L"], color: "blue" }
│
└── COLLECTION: "orders"
    ├── Document: { user_id: "...", items: [...], total: 1028 }
    └── Document: { user_id: "...", items: [...], coupon: "SAVE10" }
```

### Data Modeling: Embed vs Reference

The most important decision in document databases:

```
EMBEDDING (Denormalized):                    REFERENCING (Normalized):

┌──────────────────────────────┐            ┌─────────────────────────────┐
│ {                            │            │ USER DOCUMENT:              │
│   "name": "Alice",          │            │ {                           │
│   "address": {              │            │   "name": "Alice",          │
│     "city": "Mumbai"  ◄─── EMBEDDED     │   "address_id": "addr_1"   │ ──┐
│   },                        │            │ }                           │   │
│   "orders": [               │            └─────────────────────────────┘   │
│     {"item": "Laptop"} ◄── EMBEDDED                                       │
│   ]                         │            ┌─────────────────────────────┐   │
│ }                           │            │ ADDRESS DOCUMENT:           │   │
└──────────────────────────────┘            │ {                           │◀──┘
                                            │   "_id": "addr_1",          │
✅ One read gets everything                 │   "city": "Mumbai"          │
✅ Better read performance                  │ }                           │
❌ Document can get huge                    └─────────────────────────────┘
❌ Duplicate data                           
                                            ✅ No data duplication
                                            ✅ Documents stay small
                                            ❌ Requires multiple reads
                                            ❌ No JOIN (app-level join)
```

**Rule of thumb:**
- **Embed** when data is read together and doesn't change independently
- **Reference** when data is shared, large, or changes frequently

---

## How It Works Internally

### MongoDB Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        MongoDB Server                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │  Query Engine   │    │  Aggregation    │                     │
│  │  (Parse, Plan,  │    │  Framework      │                     │
│  │   Execute)      │    │  (Pipelines)    │                     │
│  └────────┬────────┘    └────────┬────────┘                     │
│           │                      │                               │
│           ▼                      ▼                               │
│  ┌─────────────────────────────────────────┐                    │
│  │         Storage Engine (WiredTiger)       │                   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ │                   │
│  │  │ In-Memory│ │  Journal │ │ B-Tree   │ │                   │
│  │  │  Cache   │ │  (WAL)   │ │ Indexes  │ │                   │
│  │  └──────────┘ └──────────┘ └──────────┘ │                   │
│  └─────────────────────────────────────────┘                    │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────────────────────────────┐                    │
│  │         Disk (Data Files + Indexes)      │                   │
│  └─────────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

### WiredTiger Storage Engine

MongoDB uses **WiredTiger** (since v3.2):

```
Document Written:
│
├──▶ Journal (WAL) — Written first for crash safety
│
├──▶ In-Memory Cache — Kept for fast reads
│     (configurable: default 50% of RAM - 1GB)
│
└──▶ Background checkpoint — Flushes to disk every 60 seconds

Data format on disk: BSON (Binary JSON)
Compression: Snappy (default) or Zstd (better ratio)
Index structure: B-Tree (default) or LSM-Tree
```

### BSON Format (Binary JSON)

MongoDB stores documents in **BSON**, not JSON:

```
JSON:                          BSON:
{                              ┌────────────────────────┐
  "name": "Alice",            │ Length: 47 bytes       │
  "age": 28                   │ Type: 0x02 (string)    │
}                             │ Name: "name"           │
                              │ Value: "Alice"         │
Stored as: Text               │ Type: 0x10 (int32)    │
Size: Variable                │ Name: "age"            │
Parse: Slow                   │ Value: 28              │
                              │ End: 0x00              │
                              └────────────────────────┘
                              Stored as: Binary
                              Size: Fixed structure
                              Parse: Fast (type-aware)
```

**BSON advantages over JSON:**
- Supports additional types (Date, Binary, ObjectId, Decimal128)
- Length-prefixed → skip fields without parsing
- Faster traversal

### Sharding in MongoDB

```
┌─────────────────────────────────────────────────────┐
│                  mongos (Router)                      │
│         Knows which shard has which data             │
└──────┬──────────────────┬──────────────────┬────────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Shard 1    │   │  Shard 2    │   │  Shard 3    │
│  (Replica   │   │  (Replica   │   │  (Replica   │
│   Set)      │   │   Set)      │   │   Set)      │
│             │   │             │   │             │
│ Users A-G   │   │ Users H-P   │   │ Users Q-Z   │
│ Primary     │   │ Primary     │   │ Primary     │
│ Secondary   │   │ Secondary   │   │ Secondary   │
│ Secondary   │   │ Secondary   │   │ Secondary   │
└─────────────┘   └─────────────┘   └─────────────┘

Config Servers (metadata about which chunk is where)
```

**Shard key** determines data distribution. Choose wisely:
- Good shard key: `user_id` (even distribution)
- Bad shard key: `country` (most data in a few shards)

---

## Code Examples

### Python (pymongo)

```python
from pymongo import MongoClient
from bson.objectid import ObjectId
from datetime import datetime

# Connect to MongoDB
client = MongoClient("mongodb://localhost:27017")
db = client["ecommerce"]
users = db["users"]

# INSERT — flexible schema, no predefined structure
user = {
    "name": "Alice Johnson",
    "email": "alice@example.com",
    "joined": datetime.now(),
    "address": {
        "street": "123 Main St",
        "city": "Mumbai",
        "country": "India"
    },
    "interests": ["technology", "gaming"],
    "orders_count": 0
}
result = users.insert_one(user)
print(f"Inserted ID: {result.inserted_id}")

# QUERY — find with filters, projections
premium_users = users.find(
    {"orders_count": {"$gte": 10}},    # Filter: 10+ orders
    {"name": 1, "email": 1, "_id": 0}  # Projection: only these fields
).sort("orders_count", -1).limit(10)

# UPDATE — atomic operations on nested fields
users.update_one(
    {"email": "alice@example.com"},
    {
        "$set": {"address.city": "Bangalore"},      # Update nested field
        "$push": {"interests": "cooking"},          # Add to array
        "$inc": {"orders_count": 1}                 # Increment counter
    }
)

# AGGREGATION PIPELINE — powerful multi-stage processing
pipeline = [
    {"$match": {"joined": {"$gte": datetime(2024, 1, 1)}}},
    {"$group": {
        "_id": "$address.city",
        "count": {"$sum": 1},
        "avg_orders": {"$avg": "$orders_count"}
    }},
    {"$sort": {"count": -1}},
    {"$limit": 5}
]
city_stats = list(users.aggregate(pipeline))
for city in city_stats:
    print(f"{city['_id']}: {city['count']} users, avg {city['avg_orders']:.1f} orders")

# INDEX creation for performance
users.create_index("email", unique=True)
users.create_index([("address.city", 1), ("orders_count", -1)])
```

### Java (MongoDB Driver)

```java
import com.mongodb.client.*;
import com.mongodb.client.model.*;
import org.bson.Document;
import org.bson.conversions.Bson;
import java.util.*;

public class MongoDBExample {
    public static void main(String[] args) {
        MongoClient client = MongoClients.create("mongodb://localhost:27017");
        MongoDatabase db = client.getDatabase("ecommerce");
        MongoCollection<Document> products = db.getCollection("products");

        // Insert product with nested variants
        Document product = new Document("name", "iPhone 15")
            .append("brand", "Apple")
            .append("price", 999.99)
            .append("specs", new Document("ram", "8GB").append("storage", "256GB"))
            .append("variants", Arrays.asList(
                new Document("color", "Black").append("stock", 150),
                new Document("color", "Blue").append("stock", 80)
            ))
            .append("tags", Arrays.asList("smartphone", "premium"));

        products.insertOne(product);

        // Query: Find products with specific nested criteria
        Bson filter = Filters.and(
            Filters.gte("price", 500),
            Filters.eq("specs.ram", "8GB"),
            Filters.in("tags", "premium")
        );

        FindIterable<Document> results = products.find(filter)
            .sort(Sorts.descending("price"))
            .limit(10);

        for (Document doc : results) {
            System.out.printf("Product: %s ($%.2f)%n",
                doc.getString("name"), doc.getDouble("price"));
        }

        // Aggregation: Average price by brand
        List<Bson> pipeline = Arrays.asList(
            Aggregates.group("$brand",
                Accumulators.avg("avgPrice", "$price"),
                Accumulators.sum("count", 1)),
            Aggregates.sort(Sorts.descending("avgPrice"))
        );

        for (Document doc : products.aggregate(pipeline)) {
            System.out.printf("Brand: %s, Avg: $%.2f, Count: %d%n",
                doc.getString("_id"), doc.getDouble("avgPrice"), doc.getInteger("count"));
        }
    }
}
```

---

## Infrastructure Examples

### MongoDB Replica Set (Docker Compose)

```yaml
version: '3.8'
services:
  mongo-primary:
    image: mongo:7
    command: mongod --replSet rs0 --bind_ip_all
    ports:
      - "27017:27017"
    volumes:
      - mongo-primary-data:/data/db

  mongo-secondary1:
    image: mongo:7
    command: mongod --replSet rs0 --bind_ip_all
    ports:
      - "27018:27017"
    volumes:
      - mongo-secondary1-data:/data/db

  mongo-secondary2:
    image: mongo:7
    command: mongod --replSet rs0 --bind_ip_all
    ports:
      - "27019:27017"
    volumes:
      - mongo-secondary2-data:/data/db

volumes:
  mongo-primary-data:
  mongo-secondary1-data:
  mongo-secondary2-data:
```

### MongoDB Atlas (AWS - Terraform)

```hcl
resource "mongodbatlas_cluster" "main" {
  project_id = var.atlas_project_id
  name       = "production"

  provider_name               = "AWS"
  provider_region_name        = "AP_SOUTH_1"  # Mumbai
  provider_instance_size_name = "M30"

  cluster_type = "REPLICASET"
  replication_specs {
    num_shards = 1
    regions_config {
      region_name     = "AP_SOUTH_1"
      electable_nodes = 3
      priority        = 7
      read_only_nodes = 2
    }
  }

  auto_scaling_compute_enabled = true
  auto_scaling_compute_scale_down_enabled = true
}
```

---

## Real-World Example

### Uber — Document Store for Trip Data

Uber stores trip data in a document model:

```json
{
  "_id": "trip_abc123",
  "rider": {
    "id": "user_456",
    "name": "Rahul",
    "rating": 4.8
  },
  "driver": {
    "id": "driver_789",
    "name": "Amit",
    "vehicle": {"make": "Maruti", "model": "Swift", "plate": "MH-01-XX-1234"}
  },
  "route": {
    "pickup": {"lat": 19.076, "lng": 72.877, "address": "Andheri Station"},
    "dropoff": {"lat": 19.021, "lng": 72.856, "address": "Bandra"},
    "distance_km": 8.5,
    "duration_min": 25
  },
  "pricing": {
    "base_fare": 50,
    "distance_charge": 85,
    "surge_multiplier": 1.5,
    "total": 202.50,
    "currency": "INR"
  },
  "status": "completed",
  "timeline": [
    {"event": "requested", "timestamp": "2024-01-15T10:00:00Z"},
    {"event": "driver_assigned", "timestamp": "2024-01-15T10:01:30Z"},
    {"event": "pickup", "timestamp": "2024-01-15T10:08:00Z"},
    {"event": "dropoff", "timestamp": "2024-01-15T10:33:00Z"}
  ]
}
```

**Why document store works here:**
- Each trip is self-contained
- Trip data is read as a whole unit
- Schema varies (pool rides have multiple riders, premium rides have extra fields)
- No complex joins needed — one read per trip

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Embedding everything | Documents grow to 16MB limit, slow reads | Reference large/growing arrays |
| No indexes | Full collection scans | Create indexes for query patterns |
| Unbounded arrays | Document grows forever (e.g., all user posts) | Use separate collection for growing data |
| Using `$lookup` extensively | Effectively doing JOINs (defeats the purpose) | Redesign data model or use SQL |
| Not using aggregation pipeline | Complex logic in application code | Pipeline runs IN the database (faster) |
| Ignoring write concerns | Data loss on crash | Use `w: "majority"` for important writes |
| Schema-less = no planning | Inconsistent data, hard to query | Use schema validation rules |

---

## When to Use / When NOT to Use

### ✅ Use Document Stores When:
- Data is naturally hierarchical (user profiles, product catalogs, blog posts)
- Schema varies between documents (CMS content, IoT device configs)
- You read/write entire entities at once (not individual fields)
- Rapid prototyping — schema evolves frequently
- Horizontal scaling is needed beyond single-server capacity
- You need flexible, nested data without complex JOINs

### ❌ Do NOT Use Document Stores When:
- Data has complex relationships (many-to-many between entities)
- You need ACID transactions across multiple documents/collections
- Heavy aggregation across different entity types (analytics)
- Data integrity constraints are critical (financial systems)
- You frequently need just one field from a large document
- Your queries look like SQL JOINs — that's a sign you need SQL

---

## Key Takeaways

1. **Document stores** keep related data together in one document — one read gets everything, no JOINs needed.
2. **Embed vs Reference** is the most important modeling decision — embed for data read together, reference for shared/large data.
3. **MongoDB uses BSON** (Binary JSON) internally for efficient storage and fast field traversal.
4. **WiredTiger** provides document-level locking, compression, and journaling for crash safety.
5. **Aggregation pipelines** are incredibly powerful — think of them as multi-stage data processing inside the database.
6. **16MB document limit** in MongoDB — design around it (don't store unbounded arrays).
7. **MongoDB supports ACID transactions** since v4.0, but they have performance overhead — design your schema to minimize the need for multi-document transactions.

---

## What's Next?

Next, we'll explore **Key-Value Stores (Redis, DynamoDB)** (Chapter 9.4), the simplest and fastest NoSQL databases, perfect for caching, sessions, and real-time leaderboards.
