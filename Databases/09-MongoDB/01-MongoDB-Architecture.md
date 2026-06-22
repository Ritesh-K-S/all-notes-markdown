# 🍃 Chapter 3B.1 — MongoDB Architecture & Concepts

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3–4 hours
> **Prerequisites:** Chapter 3A.1 (NoSQL Overview), Chapter 1.1 (What is a Database?)

> **"MongoDB didn't just challenge SQL — it rewrote the rules of how developers think about data."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why MongoDB exists** and the problem it solved
- Know the **complete architecture** — from client to disk
- Think in **Documents & Collections** instead of rows & tables
- Understand **BSON** — the secret sauce behind MongoDB's speed
- Grasp **Replica Sets** for high availability (your data never sleeps)
- Get a bird's-eye view of **Sharding** for horizontal scaling
- Know the **WiredTiger** storage engine inside-out

---

## 1. Why MongoDB? — The Origin Story

### The Year is 2007...

Dwight Merriman and Eliot Horowitz are building a massive web platform at DoubleClick (now Google). They keep hitting the same wall:

```
❌ The Relational Nightmare:
   → Schema changes require ALTER TABLE on 500M rows → 6 hours downtime
   → JOINs across 12 tables for a single page load → slow
   → Scaling? Add bigger server. Budget? Gone.
   → Different data shapes? Force everything into rigid rows/columns
```

They thought: **"What if the database just stored data the way applications use it?"**

MongoDB was born. The name comes from "hu**mongo**us" — because it was built for **humongous** amounts of data.

### What Makes MongoDB Different?

| SQL World (Relational) | MongoDB World (Document) |
|------------------------|--------------------------|
| Database | Database |
| Table | **Collection** |
| Row | **Document** (JSON/BSON) |
| Column | **Field** |
| Primary Key | **`_id`** (auto-generated if not provided) |
| JOIN | **$lookup** (or embed the data!) |
| Schema (rigid) | **Schema-flexible** (each doc can differ) |
| ALTER TABLE | **Just insert the new shape** — no migration |

> 💡 **Key Insight**: In MongoDB, you don't design tables — you design **documents** that mirror how your application actually uses the data.

---

## 2. The Document Model — Think JSON, Not Rows

### What is a Document?

A document is a **self-describing, hierarchical data structure** — essentially a JSON object stored in the database.

```json
// ── A SQL row would be FLAT ──
// id=1, name="Ritesh", age=28, city="Delhi", skill1="Python", skill2="MongoDB"

// ── A MongoDB document is RICH ──
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Ritesh",
  "age": 28,
  "address": {                          // ← Nested object (embedded document)
    "city": "Delhi",
    "state": "Delhi",
    "pin": "110001"
  },
  "skills": ["Python", "MongoDB", "Node.js"],  // ← Array
  "experience": [                       // ← Array of objects!
    {
      "company": "Google",
      "role": "Backend Engineer",
      "years": 3
    },
    {
      "company": "Amazon",
      "role": "SDE-2",
      "years": 2
    }
  ],
  "isActive": true,
  "joinedAt": ISODate("2024-01-15T10:30:00Z")
}
```

### SQL vs MongoDB — Same Data, Different World

```
SQL Approach (3 tables, 2 JOINs):
┌──────────────────┐     ┌───────────────────┐     ┌──────────────────┐
│     USERS        │     │    ADDRESSES       │     │   EXPERIENCES    │
├──────────────────┤     ├───────────────────┤     ├──────────────────┤
│ id  │ name │ age │     │ user_id │ city    │     │ user_id│ company │
│  1  │Ritesh│  28 │     │    1    │ Delhi   │     │   1   │ Google  │
└──────────────────┘     └───────────────────┘     │   1   │ Amazon  │
                                                    └──────────────────┘
To get Ritesh's profile: SELECT ... FROM users 
                         JOIN addresses ON ...
                         JOIN experiences ON ...   ← 2 JOINs!

MongoDB Approach (1 document, 0 JOINs):
┌─────────────────────────────────────────┐
│  {                                       │
│    "name": "Ritesh",                     │
│    "age": 28,                            │
│    "address": { "city": "Delhi" },       │  ← Everything in ONE document
│    "experience": [                       │
│      { "company": "Google" },            │
│      { "company": "Amazon" }             │
│    ]                                     │
│  }                                       │
└─────────────────────────────────────────┘
To get Ritesh's profile: db.users.findOne({ name: "Ritesh" })  ← Done!
```

> 💡 **Pro Tip**: The golden rule of MongoDB — **"Data that is accessed together should be stored together."**

---

## 3. BSON — Binary JSON (The Secret Sauce)

### Why Not Just Store Plain JSON?

JSON is great for humans but **terrible for databases**:

```
JSON Problems:
❌ Text-based → Slow to parse
❌ No date type → Everything is a string
❌ No integer vs float distinction
❌ No binary data support
❌ Can't efficiently seek to a field without reading the whole thing
```

MongoDB solved this with **BSON** (Binary JSON):

```
JSON  →  Human-readable text     →  { "age": 28 }
BSON  →  Binary-encoded format   →  \x10 age \x00 \x1c\x00\x00\x00

┌─────────────────────────────────────────────────────────┐
│                    BSON vs JSON                          │
├─────────────────┬──────────────┬─────────────────────────┤
│  Feature        │  JSON        │  BSON                   │
├─────────────────┼──────────────┼─────────────────────────┤
│  Format         │  Text        │  Binary                 │
│  Parse Speed    │  Slow        │  🚀 Very Fast           │
│  Size           │  Smaller     │  Slightly larger*       │
│  Data Types     │  6 types     │  18+ types              │
│  Date Support   │  ❌ String   │  ✅ Native ISODate      │
│  Binary Data    │  ❌ No       │  ✅ BinData type        │
│  ObjectId       │  ❌ No       │  ✅ Native 12-byte ID   │
│  int32/int64    │  ❌ Number   │  ✅ Distinct types      │
│  Decimal128     │  ❌ No       │  ✅ Exact decimals      │
│  Traversal      │  Sequential  │  Length-prefix skipping  │
├─────────────────┴──────────────┴─────────────────────────┤
│  * BSON is slightly larger because it stores type info    │
│    and field lengths — but this makes it MUCH faster to   │
│    traverse and query.                                    │
└──────────────────────────────────────────────────────────┘
```

### BSON Data Types You Must Know

| Type | Example | When To Use |
|------|---------|-------------|
| `String` | `"Ritesh"` | Names, descriptions |
| `Int32` | `NumberInt(42)` | Small integers |
| `Int64` | `NumberLong(9999999999)` | Large integers, timestamps |
| `Double` | `3.14` | Floating point numbers |
| `Decimal128` | `NumberDecimal("19.99")` | Money (exact precision!) |
| `Boolean` | `true / false` | Flags, toggles |
| `Date` | `ISODate("2024-01-15")` | Timestamps |
| `ObjectId` | `ObjectId("507f1f77...")` | Default `_id` type |
| `Array` | `["a", "b", "c"]` | Lists, tags |
| `Object` | `{ "key": "val" }` | Embedded documents |
| `Null` | `null` | Missing/unknown value |
| `BinData` | `BinData(0, "base64...")` | Images, files, encrypted data |
| `Regex` | `/pattern/i` | Pattern matching |
| `Timestamp` | `Timestamp(1, 1)` | Internal replication use |

### The ObjectId — MongoDB's Auto-Generated Primary Key

Every document needs an `_id`. If you don't provide one, MongoDB generates an **ObjectId**:

```
ObjectId = 12 bytes (24 hex characters)

  507f1f77  bcf86c   d799  439011
  ┬──────── ┬─────── ┬──── ┬──────
  │         │        │     │
  │         │        │     └─ Counter (3 bytes) — incrementing
  │         │        └─ Process ID (2 bytes)** — which process
  │         └─ Random value (5 bytes) — unique per machine
  └─ Timestamp (4 bytes) — when it was created
  
  ** Note: In MongoDB 3.4+, the 5 bytes represent a random value
     instead of separate machine ID + PID.

Why this design?
✅ Globally unique without coordination (no auto-increment locks!)
✅ Contains creation timestamp (extract it: ObjectId.getTimestamp())
✅ Roughly time-ordered (good for default sorting)
✅ 12 bytes < 16 bytes (UUID) = more compact
```

> 💡 **Pro Tip**: You can extract the creation time from any ObjectId:
> ```js
> ObjectId("507f1f77bcf86cd799439011").getTimestamp()
> // Returns: ISODate("2012-10-17T20:46:31Z")
> ```
> Free timestamp without an extra field!

---

## 4. MongoDB Architecture — The Complete Picture

### 4.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     MongoDB Architecture                             │
│                                                                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                             │
│  │  App 1  │  │  App 2  │  │  App 3  │    ← Applications            │
│  └────┬────┘  └────┬────┘  └────┬────┘                              │
│       │            │            │                                    │
│       ▼            ▼            ▼                                    │
│  ┌─────────────────────────────────────┐                             │
│  │     MongoDB Driver (Client)         │   ← Language-specific       │
│  │  (Node.js, Python, Java, C#, etc.)  │     driver library          │
│  └────────────────┬────────────────────┘                             │
│                   │  Wire Protocol (TCP)                              │
│                   ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐             │
│  │              mongod (Server Process)                 │             │
│  │  ┌──────────────────────────────────────────────┐   │             │
│  │  │           Query Engine                        │   │             │
│  │  │  ┌──────────┐ ┌───────────┐ ┌─────────────┐  │   │             │
│  │  │  │  Parser  │→│ Optimizer │→│  Executor   │  │   │             │
│  │  │  └──────────┘ └───────────┘ └─────────────┘  │   │             │
│  │  └──────────────────────────────────────────────┘   │             │
│  │  ┌──────────────────────────────────────────────┐   │             │
│  │  │        Storage Engine (WiredTiger)            │   │             │
│  │  │  ┌──────────┐ ┌───────┐ ┌─────────────────┐  │   │             │
│  │  │  │  Cache   │ │Journal│ │ Data Files (.wt) │  │   │             │
│  │  │  │(In-Mem)  │ │ (WAL) │ │  (On Disk)       │  │   │             │
│  │  │  └──────────┘ └───────┘ └─────────────────┘  │   │             │
│  │  └──────────────────────────────────────────────┘   │             │
│  └─────────────────────────────────────────────────────┘             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Key Components Explained

| Component | What It Does |
|-----------|-------------|
| **`mongod`** | The core database daemon/process — handles data requests, manages storage, performs operations |
| **`mongos`** | Router process for **sharded clusters** — directs queries to the right shard |
| **Driver** | Language library (pymongo, mongoose, etc.) that your app uses to talk to MongoDB |
| **Wire Protocol** | Binary protocol over TCP/IP for client-server communication |
| **Query Engine** | Parses your query → plans the optimal path → executes it |
| **Storage Engine** | Actually reads/writes data to disk — WiredTiger is the default since MongoDB 3.2 |

---

## 5. WiredTiger — The Engine Under the Hood

> Since MongoDB 3.2, **WiredTiger** is the default (and only production) storage engine. Understanding it = understanding MongoDB performance.

### 5.1 How WiredTiger Works

```
                    WiredTiger Architecture

  ┌────────────────────────────────────────────────────┐
  │                   WiredTiger Cache                  │
  │            (configurable, default 50% RAM)          │
  │                                                     │
  │   ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
  │   │ B+Tree for  │  │ B+Tree for  │  │ B+Tree    │ │
  │   │ Collection  │  │  Index 1    │  │ Index 2   │ │
  │   │   Data      │  │             │  │           │ │
  │   └──────┬──────┘  └──────┬──────┘  └─────┬─────┘ │
  │          │                │                │       │
  └──────────┼────────────────┼────────────────┼───────┘
             │                │                │
             ▼                ▼                ▼
  ┌────────────────────────────────────────────────────┐
  │              Checkpoint System                      │
  │  (Periodic snapshots — every 60 sec or 2GB journal) │
  └────────────────────────┬───────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │ Data Files   │ │ Index Files  │ │ Journal Files│
  │ (.wt)        │ │ (.wt)        │ │ (WAL)        │
  │              │ │              │ │              │
  │ collection-  │ │ index-xxx.wt │ │ Write-Ahead  │
  │ xxx.wt       │ │              │ │ Log for      │
  │              │ │              │ │ crash safety  │
  └──────────────┘ └──────────────┘ └──────────────┘
          ▲                                 │
          │    On crash recovery,           │
          └────── replay journal ◄──────────┘
```

### 5.2 Key WiredTiger Features

| Feature | What It Does | Why It Matters |
|---------|-------------|----------------|
| **Document-Level Locking** | Only locks the specific document being modified | 1000 users can write to the same collection simultaneously |
| **Compression** | Snappy (default), zlib, zstd | Saves 60–80% disk space |
| **In-Memory Cache** | Hot data stays in RAM | Reads are RAM-speed, not disk-speed |
| **Checkpoints** | Periodic consistent snapshots to disk | Crash recovery without replaying entire journal |
| **Journaling (WAL)** | Write-Ahead Log — writes to journal before data files | Zero data loss on crash |
| **MVCC** | Multi-Version Concurrency Control | Readers don't block writers, writers don't block readers |

### 5.3 Write Path (What Happens When You Insert)

```
db.users.insertOne({ name: "Ritesh", age: 28 })

Step 1: Driver sends BSON document to mongod
        │
Step 2: ▼ Write to JOURNAL (WAL) on disk
        │  → This is your safety net
        │  → If crash happens here, journal replays on restart
        │
Step 3: ▼ Write to WiredTiger CACHE (in RAM)
        │  → Fast! This is where reads will find it
        │
Step 4: ▼ Eventually... CHECKPOINT flushes to DATA FILES
        │  → Every 60 seconds (or 2GB of journal data)
        │  → Writes cache → disk in a consistent snapshot
        │
Step 5: ✅ Acknowledged to client
        │  → "Your write is safe" (depends on write concern)

Timeline:
  Client ──write──▶ Journal (disk) ──▶ Cache (RAM) ──▶ Acknowledge
                                           │
                                     (background)
                                           │
                                     Checkpoint ──▶ Data Files (disk)
```

### 5.4 Read Path (What Happens When You Query)

```
db.users.findOne({ name: "Ritesh" })

Step 1: Query Engine parses and optimizes the query
        │
Step 2: ▼ Check: Is there an INDEX on "name"?
        │
        ├── YES → Use index to find document location → O(log n)
        │
        └── NO  → COLLECTION SCAN (read every document) → O(n) 🐌
        │
Step 3: ▼ Look for document in WiredTiger CACHE (RAM)
        │
        ├── CACHE HIT  → Return immediately ⚡ (~0.1ms)
        │
        └── CACHE MISS → Read from DISK, load into cache, then return
        │
Step 4: ✅ Return BSON document → Driver converts to language object
```

> 💡 **Pro Tip**: Monitor cache hit ratio with `db.serverStatus().wiredTiger.cache`. If `"bytes read into cache"` is consistently high, you need more RAM.

---

## 6. Collections & Databases — The Organizational Structure

### 6.1 The Hierarchy

```
MongoDB Instance (mongod)
  │
  ├── Database: "ecommerce"
  │     ├── Collection: "users"
  │     │     ├── Document: { _id: 1, name: "Ritesh", ... }
  │     │     ├── Document: { _id: 2, name: "Priya", ... }
  │     │     └── Document: { _id: 3, name: "Amit", ... }
  │     │
  │     ├── Collection: "products"
  │     │     ├── Document: { _id: 1, name: "iPhone 15", price: 79999, ... }
  │     │     └── Document: { _id: 2, name: "MacBook Pro", specs: { ... }, ... }
  │     │
  │     └── Collection: "orders"
  │           └── Document: { _id: 1, userId: 1, items: [...], ... }
  │
  ├── Database: "analytics"
  │     └── Collection: "events"
  │
  ├── Database: "admin"        ← System DB (authentication, authorization)
  ├── Database: "local"        ← Instance-specific data (replication oplog)
  └── Database: "config"       ← Sharding metadata (sharded clusters only)
```

### 6.2 Schema-Flexible ≠ Schema-Less

This is the **#1 misconception** about MongoDB:

```
❌ WRONG: "MongoDB has no schema — store anything anywhere!"

✅ RIGHT: "MongoDB has a FLEXIBLE schema — documents in a collection
          CAN have different fields, but SHOULD follow a consistent shape."
```

**In practice**, you should:

```
// ✅ GOOD — Consistent shape, some optional fields
{ _id: 1, name: "Ritesh", age: 28, email: "r@x.com" }
{ _id: 2, name: "Priya",  age: 25, email: "p@x.com", phone: "+91..." }
                                                        ↑ optional field, OK

// ❌ BAD — Total chaos in the same collection
{ _id: 1, name: "Ritesh", age: 28 }
{ _id: 2, product: "iPhone", price: 999 }     // ← This is a PRODUCT, not a USER!
{ _id: 3, event: "click", timestamp: "..." }  // ← This is an EVENT!
```

### 6.3 Schema Validation (Enforce Rules When You Need Them)

MongoDB lets you add **JSON Schema validation** to collections:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "age"],
      properties: {
        name: {
          bsonType: "string",
          description: "must be a string and is required"
        },
        email: {
          bsonType: "string",
          pattern: "^.+@.+$",
          description: "must be a valid email"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150,
          description: "must be between 0 and 150"
        }
      }
    }
  }
})

// Now try inserting invalid data:
db.users.insertOne({ name: 123, email: "not-email" })
// ❌ ERROR: Document failed validation
```

> 💡 **Pro Tip**: Use `validationLevel: "moderate"` to only validate inserts and updates (not existing docs), and `validationAction: "warn"` to log warnings instead of rejecting writes during migration periods.

---

## 7. Replica Sets — Your Data Never Sleeps

### 7.1 What is a Replica Set?

A Replica Set is a group of `mongod` instances that maintain the **same data set**. It provides:
- **High Availability** — if one server dies, another takes over automatically
- **Data Redundancy** — your data exists on multiple machines
- **Read Scaling** — spread read load across replicas

```
                     MongoDB Replica Set

  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │   ┌─────────────┐    Heartbeat     ┌────────────┐│
  │   │   PRIMARY    │◄──────────────►│ SECONDARY  ││
  │   │             │    (every 2s)    │    #1      ││
  │   │  Reads ✅   │                  │            ││
  │   │  Writes ✅  │   ┌──────────┐   │  Reads ✅  ││
  │   │             │──►│  Oplog   │──►│  Writes ❌ ││
  │   └─────────────┘   │(capped   │   └────────────┘│
  │         ▲           │collection)│                  │
  │         │           └──────┬───┘                  │
  │    Application             │        ┌────────────┐│
  │    connects to             └───────►│ SECONDARY  ││
  │    PRIMARY for                      │    #2      ││
  │    writes                           │            ││
  │                                     │  Reads ✅  ││
  │                                     │  Writes ❌ ││
  │                                     └────────────┘│
  │                                                   │
  └──────────────────────────────────────────────────┘

  Minimum recommended: 3 members (1 Primary + 2 Secondaries)
```

### 7.2 How Replication Works — The Oplog

```
The OPLOG (Operations Log):
→ A special CAPPED collection on the Primary
→ Records every write operation
→ Secondaries TAIL the oplog and replay operations

Timeline:
  t1: Client writes { name: "Ritesh" } to PRIMARY
  t2: PRIMARY records in oplog: { op: "i", ns: "mydb.users", o: {...} }
  t3: SECONDARY #1 reads oplog entry, applies write
  t4: SECONDARY #2 reads oplog entry, applies write

  Result: All 3 nodes have identical data!

Oplog Entry Example:
{
  "ts": Timestamp(1625000000, 1),    // When
  "op": "i",                          // Operation: i=insert, u=update, d=delete
  "ns": "ecommerce.users",            // Namespace (db.collection)
  "o": {                              // The actual document/operation
    "_id": ObjectId("..."),
    "name": "Ritesh",
    "age": 28
  }
}
```

### 7.3 Automatic Failover — The Election

```
What happens when PRIMARY dies?

  ┌───────────┐         ┌───────────┐         ┌───────────┐
  │  PRIMARY  │         │SECONDARY 1│         │SECONDARY 2│
  │   💀 DOWN │         │           │         │           │
  └───────────┘         └─────┬─────┘         └─────┬─────┘
                              │                     │
                        "Primary is dead!"    "I agree!"
                              │                     │
                              ▼                     ▼
                     ┌─────────────────────────────────┐
                     │         ELECTION BEGINS          │
                     │                                  │
                     │  • Secondaries check: who has    │
                     │    the MOST up-to-date data?     │
                     │  • Nodes vote (majority wins)    │
                     │  • Election takes ~12 seconds    │
                     └──────────────┬──────────────────┘
                                   │
                                   ▼
  ┌───────────┐         ┌───────────┐         ┌───────────┐
  │   DOWN    │         │ ★ NEW     │         │SECONDARY  │
  │  (heals   │         │  PRIMARY  │         │           │
  │   later → │         │           │         │           │
  │ secondary)│         │ Reads ✅  │         │ Reads ✅  │
  │           │         │ Writes ✅ │         │ Writes ❌ │
  └───────────┘         └───────────┘         └───────────┘

  ✅ Zero manual intervention
  ✅ Application reconnects automatically (via driver)
  ✅ Total downtime: ~12 seconds (election time)
```

### 7.4 Read Preferences

| Read Preference | Where Reads Go | Use Case |
|----------------|---------------|----------|
| `primary` (default) | Primary only | Strongest consistency |
| `primaryPreferred` | Primary, fallback to secondary | Mostly consistent |
| `secondary` | Secondary only | Offload reads from primary |
| `secondaryPreferred` | Secondary, fallback to primary | Analytics queries |
| `nearest` | Lowest network latency member | Geo-distributed apps |

> ⚠️ **Warning**: Reading from secondaries means you might read **stale data** (replication lag). For critical reads (account balance, inventory), always use `primary`.

---

## 8. Sharding — Horizontal Scaling for Massive Data

### 8.1 The Problem: Vertical Scaling Has Limits

```
Vertical Scaling (Scale Up):
  Year 1: 10GB data  → 16GB RAM server     ✅ Fine
  Year 2: 500GB data → 128GB RAM server    ✅ Expensive but OK
  Year 3: 5TB data   → 1TB RAM server      ❌ Doesn't exist / costs $$$
  Year 4: 50TB data  → ???                  ❌ IMPOSSIBLE

Horizontal Scaling (Scale Out) — SHARDING:
  Split data across multiple machines:
  
  50TB total data:
  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │ 10TB │ │ 10TB │ │ 10TB │ │ 10TB │ │ 10TB │
  │Shard1│ │Shard2│ │Shard3│ │Shard4│ │Shard5│
  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
  
  ✅ Each machine handles 1/5 of the data
  ✅ Need more capacity? Add another shard!
```

### 8.2 Sharded Cluster Architecture

```
                    Sharded Cluster Architecture

    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │  App 1   │  │  App 2   │  │  App 3   │
    └────┬─────┘  └────┬─────┘  └────┬─────┘
         │             │             │
         ▼             ▼             ▼
    ┌──────────────────────────────────────┐
    │         mongos (Query Router)         │
    │                                      │
    │  "Which shard has the data you need? │
    │   Let me route your query."          │
    └──────────────┬───────────────────────┘
                   │
         ┌─────────┼─────────┐
         │         │         │
    ┌────▼────┐┌───▼───┐┌───▼────┐
    │ Shard 1 ││Shard 2││ Shard 3│      ← Each shard is a
    │ (RS)    ││ (RS)  ││  (RS)  │         Replica Set!
    │A-F users││G-N    ││ O-Z   │
    └─────────┘└───────┘└────────┘

    ┌────────────────────────────────────┐
    │  Config Servers (Replica Set)      │
    │                                    │
    │  Stores: Shard metadata, chunk     │
    │  ranges, cluster configuration     │
    └────────────────────────────────────┘
```

### 8.3 Shard Keys — The Most Critical Decision

The **shard key** determines how data is distributed. Choose wrong, and you get **hot spots**:

```
Shard Key Examples:

✅ GOOD Shard Key: { userId: 1 }
   → Evenly distributed across shards
   → Most queries include userId (targeted queries)
   
   Shard 1: userId 1-1000
   Shard 2: userId 1001-2000
   Shard 3: userId 2001-3000
   → Equal distribution ✅

❌ BAD Shard Key: { country: 1 }
   → 80% of users might be from ONE country
   
   Shard 1: country="India"     ← 80% of data! HOT SPOT 🔥
   Shard 2: country="USA"       ← 10%
   Shard 3: country="Others"    ← 10%
   → Massively unbalanced ❌

❌ TERRIBLE Shard Key: { createdAt: 1 }
   → All new writes go to the LATEST shard
   
   Shard 1: Jan-June (old, idle)
   Shard 2: July-Dec (old, idle)
   Shard 3: Current month ← ALL writes here! 🔥💀
   → "Monotonically increasing" = anti-pattern
```

> 💡 **Pro Tip**: A shard key should have (1) **high cardinality** (many unique values), (2) **low frequency** (values well distributed), and (3) **non-monotonic** (not always increasing). Hashed shard keys solve monotonic problems.

---

## 9. MongoDB Server Components

### 9.1 The `mongod` Process

```
mongod — Core database server process

  Responsibilities:
  ├── Data storage and retrieval
  ├── Query execution
  ├── Index management
  ├── Replication (oplog management)
  ├── Authentication & authorization
  ├── Journaling for crash recovery
  └── Memory management (WiredTiger cache)

  Common startup:
  $ mongod --dbpath /data/db --port 27017 --replSet myRS

  Key Config Options:
  ┌───────────────────────┬────────────────────────────────┐
  │  Option               │  Purpose                       │
  ├───────────────────────┼────────────────────────────────┤
  │  --dbpath             │  Where data files are stored   │
  │  --port               │  Listen port (default: 27017)  │
  │  --replSet            │  Replica set name              │
  │  --bind_ip            │  Which IP to listen on         │
  │  --auth               │  Enable authentication         │
  │  --wiredTigerCacheSizeGB │ Cache size (default: 50% RAM)│
  └───────────────────────┴────────────────────────────────┘
```

### 9.2 The `mongos` Router

```
mongos — Query Router for Sharded Clusters

  → Receives client queries
  → Checks config servers: "Where is this data?"
  → Routes to correct shard(s)
  → Merges results from multiple shards if needed

  Query Routing:
  ┌──────────────────────────────────────────┐
  │  db.users.find({ userId: 42 })           │
  │                                          │
  │  mongos checks: userId 42 → Shard 1      │
  │  → Sends query ONLY to Shard 1           │
  │  → This is a "TARGETED" query ⚡         │
  └──────────────────────────────────────────┘

  ┌──────────────────────────────────────────┐
  │  db.users.find({ age: 28 })              │
  │                                          │
  │  mongos: age is NOT the shard key!       │
  │  → Must ask ALL shards (scatter/gather)  │
  │  → This is a "BROADCAST" query 🐌       │
  └──────────────────────────────────────────┘
```

---

## 10. Write Concern & Read Concern — Consistency Guarantees

### Write Concern — "How safe is my write?"

```
Write Concern Levels:

  w: 0   → "Fire and forget" — don't wait for acknowledgment
           ⚡ Fastest, ❌ No guarantee
  
  w: 1   → Wait for PRIMARY to acknowledge (DEFAULT)
           ✅ Data is in primary's memory
           ❌ Could lose if primary dies before replication
  
  w: "majority" → Wait for MAJORITY of replicas to acknowledge
           ✅ Survives failover
           🐌 Slower (must wait for replication)
  
  j: true → Wait for JOURNAL write (not just memory)
           ✅ Survives crash
           🐌 Slightly slower

  Example:
  db.orders.insertOne(
    { item: "iPhone 15", qty: 1 },
    { writeConcern: { w: "majority", j: true } }  // Maximum safety
  )
```

### Read Concern — "How fresh is my read?"

| Read Concern | Guarantee | Speed |
|-------------|-----------|-------|
| `"local"` (default) | Returns the node's latest data (might be rolled back) | ⚡ Fast |
| `"available"` | Returns available data (even orphaned docs in sharded) | ⚡ Fastest |
| `"majority"` | Returns data acknowledged by majority (durable) | 🟡 Moderate |
| `"linearizable"` | Returns the MOST recent majority-committed data | 🐌 Slowest |
| `"snapshot"` | Point-in-time consistent snapshot (for transactions) | 🟡 Moderate |

---

## 11. Quick Reference — MongoDB vs RDBMS Terminology

```
┌────────────────────────┬─────────────────────────────┐
│   RDBMS Term           │   MongoDB Term               │
├────────────────────────┼─────────────────────────────┤
│   Database             │   Database                   │
│   Table                │   Collection                 │
│   Row                  │   Document                   │
│   Column               │   Field                      │
│   Primary Key          │   _id                        │
│   Index                │   Index                      │
│   JOIN                 │   $lookup / Embedding        │
│   GROUP BY             │   $group (Aggregation)       │
│   Foreign Key          │   Reference (_id) / DBRef    │
│   Transaction          │   Multi-Document Transaction │
│   Schema / DDL         │   Validator / Flexible       │
│   View                 │   View (read-only)           │
│   Stored Procedure     │   ❌ Not supported*          │
│   Trigger              │   Change Streams / Atlas     │
│                        │   Triggers                   │
└────────────────────────┴─────────────────────────────┘
  * You can use server-side JavaScript, but it's deprecated.
    Use application logic or Atlas Functions instead.
```

---

## 🧠 Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│                   CHAPTER 3B.1 SUMMARY                        │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. MongoDB stores data as BSON DOCUMENTS (rich JSON)         │
│                                                               │
│  2. Documents live in COLLECTIONS, collections in DATABASES   │
│                                                               │
│  3. BSON > JSON: more types, faster parsing, binary encoding  │
│                                                               │
│  4. WiredTiger: document-level locking, compression,          │
│     journaling, MVCC — it's the real engine                   │
│                                                               │
│  5. Write path: Journal → Cache → (Checkpoint) → Data Files  │
│                                                               │
│  6. REPLICA SETS: 3+ nodes, automatic failover, oplog-based  │
│                                                               │
│  7. SHARDING: horizontal scaling by splitting data across     │
│     machines using a SHARD KEY                                │
│                                                               │
│  8. Write/Read Concern: tunable consistency vs performance    │
│                                                               │
│  9. Schema-flexible ≠ schema-less. Use validators!            │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|-------------|-------|
| **[3B.2 — MongoDB Installation & Setup](./02-MongoDB-Installation.md)** | Install MongoDB, connect with mongosh & Compass, set up Atlas |

---

> **"Understanding MongoDB's architecture is like knowing the map before the journey. Now that you see the landscape, the code will make perfect sense."**
