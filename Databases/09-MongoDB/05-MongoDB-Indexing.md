# ⚡ Chapter 3B.5 — MongoDB Indexing & Performance

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 3B.3 (MongoDB CRUD), Chapter 1.7 (Indexing Foundations)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why MongoDB indexes are non-negotiable** for production
- Create every type of index — **Single, Compound, Multikey, Text, Geo, Wildcard, Hashed**
- Read `explain()` output like a **MongoDB performance ninja**
- Identify and fix **slow queries** using the profiler
- Know the **ESR Rule** (Equality → Sort → Range) — the secret to compound index design
- Avoid common indexing **anti-patterns** that destroy performance

---

## 🧠 The Big Picture — Why Indexes Matter in MongoDB

MongoDB stores documents in **collections**. Without indexes, every query does a **COLLSCAN** (Collection Scan) — reading **every single document** to find matches.

```
❌ Without Index (COLLSCAN):
   db.users.find({ email: "john@example.com" })
   
   → MongoDB walks through ALL 10,000,000 documents
   → Checks each document's "email" field
   → Finds the ONE match after reading everything
   → Time: 12 seconds 🐌

✅ With Index (IXSCAN):
   db.users.createIndex({ email: 1 })
   db.users.find({ email: "john@example.com" })
   
   → MongoDB looks up "john@example.com" in the B-Tree index
   → Jumps directly to the document
   → Time: 0.5 milliseconds ⚡
   
   That's 24,000x faster. Not an exaggeration.
```

> 💡 **Rule #1:** Every query in production **MUST** be backed by an appropriate index. No exceptions.

---

## 🔑 The `_id` Index — You Already Have One

Every MongoDB collection **automatically** creates a unique index on `_id`:

```javascript
// This index exists by default — you never need to create it
// { _id: 1 }  ← unique, automatically created

// These queries ALWAYS use the _id index
db.users.find({ _id: ObjectId("507f1f77bcf86cd799439011") })  // ⚡ Fast
db.users.findOne({ _id: ObjectId("507f1f77bcf86cd799439011") }) // ⚡ Fast

// ⚠️ You CANNOT drop the _id index
db.users.dropIndex({ _id: 1 })  // ❌ Error!
```

---

## 📌 Single Field Index — The Foundation

The simplest and most common index. One field, one direction.

```javascript
// Syntax: db.collection.createIndex({ field: direction })
// direction: 1 = ascending, -1 = descending

// Index on email (ascending)
db.users.createIndex({ email: 1 })

// Index on createdAt (descending — newest first)
db.orders.createIndex({ createdAt: -1 })

// With options
db.users.createIndex(
  { email: 1 },
  { 
    unique: true,           // No duplicate emails
    name: "idx_email_uniq", // Custom name (auto-generated otherwise)
    background: true        // Build without blocking (deprecated in 4.2+, now default)
  }
)
```

### When Single Field Index Is Used

```javascript
// ✅ All of these use the { email: 1 } index:
db.users.find({ email: "john@example.com" })
db.users.find({ email: { $gt: "a", $lt: "m" } })       // Range query
db.users.find({ email: { $in: ["a@b.com", "c@d.com"] } }) // $in
db.users.find().sort({ email: 1 })                       // Sort ascending
db.users.find().sort({ email: -1 })                      // Sort descending! ✅

// Wait — descending sort works on ascending index?
// YES! MongoDB can traverse a single-field index in BOTH directions.
// Direction only matters for COMPOUND indexes.
```

### Unique Index

```javascript
// Prevent duplicate values
db.users.createIndex({ email: 1 }, { unique: true })

// Now this fails:
db.users.insertOne({ email: "john@example.com" })  // ✅ First insert
db.users.insertOne({ email: "john@example.com" })  // ❌ E11000 duplicate key error

// ⚠️ Gotcha: null values count!
// If two documents DON'T have the "email" field,
// both get email: null → duplicate key error!

// Fix: Use partial index to only index documents that HAVE the field
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { email: { $exists: true } } }
)
```

---

## 🧩 Compound Index — The Real-World Workhorse

An index on **two or more fields**. This is where 80% of your production indexes live.

```javascript
// Create a compound index
db.orders.createIndex({ customerId: 1, orderDate: -1 })
```

### The Left-Prefix Rule (Critical! Same as SQL)

```
Index: { customerId: 1, orderDate: -1, status: 1 }

This index supports queries on:
  ✅ { customerId: ... }                           (prefix)
  ✅ { customerId: ..., orderDate: ... }            (prefix)
  ✅ { customerId: ..., orderDate: ..., status: ... } (full)
  ❌ { orderDate: ... }                             (missing left prefix!)
  ❌ { status: ... }                                (missing left prefix!)
  ❌ { orderDate: ..., status: ... }                (missing left prefix!)
  ⚠️ { customerId: ..., status: ... }               (uses customerId only, skips orderDate)
```

```
    Visual: Think of it like a phone book

    ┌──────────────────────────────────────────────────────────┐
    │  Compound Index: { lastName: 1, firstName: 1, age: 1 }  │
    │                                                          │
    │  Sorted like a phone book:                               │
    │                                                          │
    │    Anderson, Alice, 25                                   │
    │    Anderson, Bob, 30                                     │
    │    Anderson, Charlie, 28                                 │
    │    Baker, Alice, 22                                      │
    │    Baker, David, 35                                      │
    │    Carter, Alice, 29                                     │
    │    Carter, Bob, 31                                       │
    │                                                          │
    │  Finding all "Anderson" → Easy! They're grouped together │
    │  Finding all "Alice" → Hard! Scattered everywhere        │
    │  That's WHY the left-prefix rule exists!                 │
    └──────────────────────────────────────────────────────────┘
```

### Sort Direction Matters in Compound Indexes!

```javascript
// Index: { price: 1, rating: -1 }

// ✅ Matches index sort direction
db.products.find().sort({ price: 1, rating: -1 })   // ✅ Uses index forward
db.products.find().sort({ price: -1, rating: 1 })   // ✅ Uses index backward

// ❌ Doesn't match (mixed from what index provides)
db.products.find().sort({ price: 1, rating: 1 })    // ❌ Can't use index for sort
db.products.find().sort({ price: -1, rating: -1 })  // ❌ Can't use index for sort
```

```
    Why? The index stores data like this:

    { price: 1, rating: -1 }
    ┌────────┬────────┐
    │ price  │ rating │
    ├────────┼────────┤
    │  10    │  5.0   │  ← Forward:  price ↑, rating ↓
    │  10    │  4.5   │
    │  10    │  3.0   │
    │  20    │  4.8   │
    │  20    │  4.2   │
    │  30    │  5.0   │
    │  30    │  2.1   │
    └────────┴────────┘
    
    Reading forward  → price ASC, rating DESC    ✅
    Reading backward → price DESC, rating ASC    ✅
    Any other combo  → NOT sorted properly       ❌
```

---

## 🔥 The ESR Rule — Secret to Compound Index Design

> **E**quality → **S**ort → **R**ange
> This is the **optimal ordering** for fields in a compound index.

```javascript
// Query:
db.orders.find({ 
  status: "shipped",              // Equality (exact match)
  orderDate: { $gte: ISODate("2024-01-01") }  // Range
}).sort({ totalAmount: -1 })       // Sort

// ❌ BAD index (Range before Sort):
db.orders.createIndex({ status: 1, orderDate: -1, totalAmount: -1 })
// MongoDB can't use index for sort after a range scan → in-memory sort! 🐌

// ✅ GOOD index (ESR Rule):
db.orders.createIndex({ status: 1, totalAmount: -1, orderDate: -1 })
//                       ↑ Equality   ↑ Sort          ↑ Range
//                       FIRST         SECOND          LAST

// Why?
// 1. Equality narrows to exact matches (small subset)
// 2. Sort is already in index order (no in-memory sort needed!)
// 3. Range further filters within the sorted results
```

```
    ESR in Action:

    Index: { status: 1, totalAmount: -1, orderDate: -1 }

    ┌──────────┬─────────────┬────────────┐
    │ status   │ totalAmount │ orderDate  │
    ├──────────┼─────────────┼────────────┤
    │ pending  │ 999         │ 2024-03-01 │
    │ pending  │ 500         │ 2024-02-15 │
    │ pending  │ 100         │ 2024-01-05 │
    │ shipped  │ 850         │ 2024-03-10 │ ← Start here (E: status=shipped)
    │ shipped  │ 700         │ 2024-02-20 │ ← Already sorted by totalAmount! (S)
    │ shipped  │ 650         │ 2024-01-15 │ ← Filter by date range (R)
    │ shipped  │ 300         │ 2024-03-05 │
    │ shipped  │ 150         │ 2024-01-01 │
    │ delivered│ 900         │ 2024-03-12 │
    └──────────┴─────────────┴────────────┘
```

---

## 🎭 Multikey Index — Indexing Arrays

When a field contains an **array**, MongoDB creates an index entry for **each element** in the array.

```javascript
// Document
{
  _id: 1,
  title: "MongoDB Guide",
  tags: ["database", "nosql", "mongodb", "tutorial"]
}

// Create index on array field
db.articles.createIndex({ tags: 1 })

// Now EACH array element is indexed:
//   "database"  → doc 1
//   "mongodb"   → doc 1
//   "nosql"     → doc 1
//   "tutorial"  → doc 1

// Queries that benefit:
db.articles.find({ tags: "nosql" })                    // ✅ Uses multikey index
db.articles.find({ tags: { $in: ["nosql", "sql"] } })  // ✅ Uses multikey index
db.articles.find({ tags: { $all: ["nosql", "mongodb"] } }) // ✅ Uses multikey index
```

### ⚠️ Multikey Compound Index Restriction

```javascript
// You CANNOT have a compound index where MORE THAN ONE field is an array!

{
  tags: ["a", "b", "c"],     // Array
  categories: ["x", "y"]      // Array
}

db.articles.createIndex({ tags: 1, categories: 1 })
// ❌ Error! Cannot index parallel arrays.
// Why? The cross-product would be explosive:
//   tags × categories = a,x | a,y | b,x | b,y | c,x | c,y
//   For large arrays, this explodes exponentially.

// ✅ OK if only ONE field in the compound index is an array:
db.articles.createIndex({ tags: 1, authorId: 1 })  // authorId is scalar → OK!
```

---

## 🔤 Text Index — Full-Text Search

> MongoDB's built-in text search (for simple cases — use Atlas Search for production).

```javascript
// Create a text index
db.articles.createIndex({ 
  title: "text", 
  content: "text" 
})

// ⚠️ Only ONE text index per collection!

// Basic text search
db.articles.find({ $text: { $search: "mongodb performance" } })
// Finds documents containing "mongodb" OR "performance"

// Phrase search (exact phrase)
db.articles.find({ $text: { $search: "\"mongodb performance\"" } })
// Finds documents containing the exact phrase "mongodb performance"

// Negation
db.articles.find({ $text: { $search: "mongodb -tutorial" } })
// Contains "mongodb" but NOT "tutorial"

// Text search with relevance score
db.articles.find(
  { $text: { $search: "mongodb indexing performance" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })  // Sort by relevance

// Weighted text index (title is 10x more important than content)
db.articles.createIndex(
  { title: "text", content: "text", tags: "text" },
  { weights: { title: 10, content: 5, tags: 1 } }
)
```

### Text Index vs Atlas Search

```
┌──────────────────────┬───────────────────────┬───────────────────────────┐
│ Feature              │ Text Index ($text)    │ Atlas Search (Lucene)     │
├──────────────────────┼───────────────────────┼───────────────────────────┤
│ Engine               │ Built-in MongoDB      │ Apache Lucene             │
│ Performance          │ Basic                 │ Production-grade          │
│ Fuzzy matching       │ ❌                    │ ✅ (typo tolerance)       │
│ Autocomplete         │ ❌                    │ ✅                        │
│ Facets               │ ❌                    │ ✅                        │
│ Synonyms             │ ❌                    │ ✅                        │
│ Highlighting         │ ❌                    │ ✅                        │
│ Custom analyzers     │ Limited               │ ✅ Full control           │
│ Scoring              │ Basic TF-IDF          │ BM25 + custom scoring     │
│ Availability         │ Self-hosted + Atlas   │ Atlas only                │
│ Use for              │ Simple search, dev    │ Production search         │
└──────────────────────┴───────────────────────┴───────────────────────────┘
```

---

## 🌍 Geospatial Indexes — Location-Based Queries

### 2dsphere Index (Earth-like sphere — use this one!)

```javascript
// Store location as GeoJSON
{
  name: "Central Park",
  location: {
    type: "Point",
    coordinates: [-73.965355, 40.782865]  // [longitude, latitude]
  }
}

// Create 2dsphere index
db.places.createIndex({ location: "2dsphere" })

// Find places within 5km of a point
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-73.98, 40.75] },
      $maxDistance: 5000  // meters
    }
  }
})

// Find places within a polygon (e.g., city boundary)
db.places.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-74.05, 40.70], [-73.90, 40.70],
          [-73.90, 40.80], [-74.05, 40.80],
          [-74.05, 40.70]  // Close the polygon
        ]]
      }
    }
  }
})

// Find places near a point, sorted by distance
db.places.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-73.98, 40.75] },
      distanceField: "distance",      // Output field with distance in meters
      maxDistance: 10000,              // 10km
      spherical: true
    }
  }
])
```

### 2d Index (Flat surface — legacy, for simple x,y coordinates)

```javascript
// For flat coordinate systems (game maps, floor plans)
db.gameObjects.createIndex({ position: "2d" })

db.gameObjects.find({
  position: { $near: [50, 50], $maxDistance: 10 }
})
```

---

## 🃏 Wildcard Index — When Schema is Unpredictable

> Added in MongoDB 4.2. For collections with **dynamic or polymorphic schemas**.

```javascript
// Problem: Different documents have different fields
{ _id: 1, product: "Laptop", specs: { ram: "16GB", cpu: "i7", gpu: "RTX 3060" } }
{ _id: 2, product: "Phone",  specs: { ram: "8GB", screen: "6.1 inch", battery: "4000mAh" } }
{ _id: 3, product: "TV",     specs: { screen: "55 inch", resolution: "4K", hdr: true } }

// You can't predict which fields will exist in specs!
// Creating individual indexes for every possible field? Impossible.

// Solution: Wildcard index
db.products.createIndex({ "specs.$**": 1 })

// Now ALL fields under specs are indexed:
db.products.find({ "specs.ram": "16GB" })        // ✅ Uses wildcard index
db.products.find({ "specs.screen": "55 inch" })  // ✅ Uses wildcard index
db.products.find({ "specs.hdr": true })           // ✅ Uses wildcard index

// Wildcard on entire document (index EVERYTHING)
db.products.createIndex({ "$**": 1 })
// ⚠️ Use with caution — large storage overhead!

// Wildcard with inclusion/exclusion
db.products.createIndex(
  { "$**": 1 },
  { wildcardProjection: { 
      specs: 1, 
      attributes: 1,
      _id: 0           // Exclude _id from wildcard
    } 
  }
)
```

### When to Use Wildcard Indexes

```
✅ USE Wildcard indexes when:
   • Schema varies across documents (polymorphic)
   • You don't know which fields will be queried
   • You're indexing IoT data, user-defined attributes, or metadata

❌ DON'T USE Wildcard indexes when:
   • You know your query patterns → use targeted compound indexes instead
   • You need compound index behavior (wildcard can't do ESR)
   • You need multikey on multiple array fields
```

---

## #️⃣ Hashed Index — For Shard Keys

> Hashed indexes compute a **hash of the field value** and index the hash.

```javascript
// Create hashed index
db.users.createIndex({ email: "hashed" })

// Used primarily for hash-based sharding
sh.shardCollection("mydb.users", { email: "hashed" })

// Properties:
// ✅ Even distribution (great for shard keys)
// ✅ Equality queries: db.users.find({ email: "john@example.com" })
// ❌ Range queries NOT supported: db.users.find({ email: { $gt: "a" } })
// ❌ Multikey (arrays) NOT supported
// ❌ Sorting NOT supported
```

---

## ⏰ TTL Index — Auto-Expiring Documents

> Documents **automatically delete** after a specified time. Perfect for sessions, logs, caches.

```javascript
// Delete documents 30 days after createdAt
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 * 24 * 30 }  // 30 days in seconds
)

// Delete at a specific time (set the field to the exact expiry time)
db.events.createIndex(
  { expireAt: 1 },
  { expireAfterSeconds: 0 }  // Delete when current time >= expireAt
)

// Insert with expiry
db.events.insertOne({
  event: "password_reset",
  expireAt: new Date("2024-12-31T23:59:59Z")  // Deletes on Dec 31
})
```

### TTL Index Rules

```
⚠️ Important constraints:
  • Field MUST be a Date type (or array of Dates — uses lowest)
  • Only works on SINGLE field indexes (not compound)
  • Deletion is NOT instant — background task runs every 60 seconds
  • Cannot create TTL on _id field
  • Cannot create TTL on a capped collection
```

---

## 🔍 Partial Index — Index Only What Matters

> Index a **subset** of documents. Smaller index = less storage, faster operations.

```javascript
// Only index active users (skip millions of inactive accounts)
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
)
// Index size: Maybe 20% of what a full index would be!

// Only index orders above $100
db.orders.createIndex(
  { customerId: 1, orderDate: -1 },
  { partialFilterExpression: { totalAmount: { $gte: 100 } } }
)

// ⚠️ Critical: Your query MUST include the filter condition!
db.users.find({ email: "john@example.com", status: "active" })  // ✅ Uses partial index
db.users.find({ email: "john@example.com" })                     // ❌ Can't use partial index!
// MongoDB can't guarantee all results are in the partial index
```

### Partial vs Sparse Index

```
┌─────────────┬──────────────────────────┬─────────────────────────────────┐
│ Feature     │ Sparse Index             │ Partial Index (Preferred)       │
├─────────────┼──────────────────────────┼─────────────────────────────────┤
│ Skips       │ Docs missing the field   │ Docs not matching expression    │
│ Flexibility │ Only null/missing check  │ Any filter ($gt, $in, $exists…) │
│ Since       │ MongoDB 2.x              │ MongoDB 3.2                     │
│ Recommended │ Legacy — avoid           │ ✅ Use this instead             │
└─────────────┴──────────────────────────┴─────────────────────────────────┘
```

---

## 🔍 explain() — Your X-Ray Vision Into Queries

> `explain()` tells you **exactly** how MongoDB executes your query.

### The Three Verbosity Levels

```javascript
// Level 1: queryPlanner (default) — shows the PLAN without executing
db.orders.find({ status: "shipped" }).explain()
db.orders.find({ status: "shipped" }).explain("queryPlanner")

// Level 2: executionStats — EXECUTES the query and shows stats
db.orders.find({ status: "shipped" }).explain("executionStats")

// Level 3: allPlansExecution — shows ALL candidate plans and why one was chosen
db.orders.find({ status: "shipped" }).explain("allPlansExecution")
```

### Reading explain() Output — The Key Fields

```javascript
db.orders.find({ 
  customerId: "C123", 
  status: "shipped" 
}).sort({ orderDate: -1 }).explain("executionStats")
```

```javascript
// Output (simplified):
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",                    // ← Final stage: fetch full documents
      "inputStage": {
        "stage": "IXSCAN",                 // ← Index scan (GOOD!)
        "keyPattern": { "customerId": 1, "status": 1, "orderDate": -1 },
        "indexName": "idx_cust_status_date",
        "direction": "forward",
        "indexBounds": {
          "customerId": ["[\"C123\", \"C123\"]"],
          "status": ["[\"shipped\", \"shipped\"]"],
          "orderDate": ["[MaxKey, MinKey]"]
        }
      }
    },
    "rejectedPlans": [ ... ]               // ← Plans that lost the competition
  },
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 42,                       // ← Documents returned
    "executionTimeMillis": 2,              // ← Time taken
    "totalKeysExamined": 42,               // ← Index entries scanned
    "totalDocsExamined": 42,               // ← Documents fetched from disk
  }
}
```

### The Golden Ratios — What to Look For

```
    ┌──────────────────────────────────────────────────────────────────┐
    │               explain() Performance Checklist                    │
    │                                                                  │
    │  🟢 PERFECT:                                                     │
    │     totalKeysExamined ≈ nReturned                                │
    │     totalDocsExamined ≈ nReturned                                │
    │     (Examined exactly what was returned — no waste!)              │
    │                                                                  │
    │  🟡 ACCEPTABLE:                                                  │
    │     totalKeysExamined > nReturned (some keys rejected)           │
    │     totalDocsExamined = nReturned                                │
    │                                                                  │
    │  🔴 BAD:                                                         │
    │     totalDocsExamined >> nReturned                                │
    │     (Fetched many docs just to discard them — missing index!)     │
    │                                                                  │
    │  🔴🔴 TERRIBLE:                                                  │
    │     stage: "COLLSCAN"                                            │
    │     (No index used — FULL collection scan!)                      │
    │                                                                  │
    │  ⚠️ WARNING SIGNS:                                               │
    │     stage: "SORT" (in-memory sort — index can't help with sort)  │
    │     stage: "SORT_KEY_GENERATOR" (building sort keys in memory)   │
    └──────────────────────────────────────────────────────────────────┘
```

### explain() Stage Types — Quick Reference

```
    Query Execution Stages (like a pipeline):

    ┌───────────────┐
    │   COLLSCAN    │  Full collection scan (NO index used) 🔴
    └───────┬───────┘
            │
    ┌───────▼───────┐
    │    IXSCAN     │  Index scan (index IS used) 🟢
    └───────┬───────┘
            │
    ┌───────▼───────┐
    │    FETCH      │  Retrieve full document from disk
    └───────┬───────┘  (skip if covered query!)
            │
    ┌───────▼───────┐
    │     SORT      │  In-memory sort 🟡 (try to avoid)
    └───────┬───────┘
            │
    ┌───────▼───────┐
    │   PROJECTION  │  Select specific fields
    └───────┬───────┘
            │
    ┌───────▼───────┐
    │    LIMIT      │  Limit results
    └───────────────┘

    Other stages:
    • COUNT_SCAN   — Counting using index only
    • IDHACK       — Query by _id (fast path)
    • SHARD_MERGE  — Merge results from shards
    • SORT_MERGE   — Merge pre-sorted streams
```

---

## 🎯 Covered Queries — The Ultimate Performance Win

> A **covered query** is answered entirely from the index — **MongoDB never touches the documents on disk**.

```javascript
// Index: { customerId: 1, status: 1, totalAmount: 1 }

// ✅ Covered query — all fields are in the index!
db.orders.find(
  { customerId: "C123", status: "shipped" },
  { _id: 0, customerId: 1, status: 1, totalAmount: 1 }  // Project ONLY indexed fields
)

// explain() shows:
// totalDocsExamined: 0  ← ZERO documents read from disk! 🚀
// totalKeysExamined: 42

// ❌ NOT covered — _id is included by default!
db.orders.find(
  { customerId: "C123" },
  { customerId: 1, totalAmount: 1 }  // _id is implicitly included!
)
// Fix: Add { _id: 0 } to the projection

// ❌ NOT covered — requesting a field NOT in the index
db.orders.find(
  { customerId: "C123" },
  { _id: 0, customerId: 1, notes: 1 }  // "notes" not in index → FETCH needed
)
```

---

## 📊 Index Intersection

> MongoDB can use **two separate indexes** to satisfy a query (but a compound index is usually better).

```javascript
// You have two separate indexes:
db.orders.createIndex({ customerId: 1 })
db.orders.createIndex({ status: 1 })

// This query MIGHT use index intersection:
db.orders.find({ customerId: "C123", status: "shipped" })

// MongoDB may:
// 1. Scan { customerId: 1 } index → get document IDs matching customerId
// 2. Scan { status: 1 } index → get document IDs matching status
// 3. Compute the INTERSECTION of both sets
// 4. Fetch only the intersected documents

// BUT a compound index is almost always better:
db.orders.createIndex({ customerId: 1, status: 1 })
// Single index scan → much more efficient
```

> 💡 **Rule of thumb:** Don't rely on index intersection. Design proper compound indexes.

---

## 🛠️ MongoDB Profiler — Finding Slow Queries

### Enable the Profiler

```javascript
// Level 0: OFF (default)
// Level 1: Slow queries only (above threshold)
// Level 2: ALL queries (⚠️ heavy — only for debugging)

// Profile queries slower than 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Profile ALL queries (for debugging)
db.setProfilingLevel(2)

// Check current profiling level
db.getProfilingStatus()
// { "was": 1, "slowms": 100 }

// Disable profiling
db.setProfilingLevel(0)
```

### Reading Profiler Output

```javascript
// View slow queries (stored in system.profile collection)
db.system.profile.find().sort({ ts: -1 }).limit(5).pretty()

// Find the slowest queries
db.system.profile.find().sort({ millis: -1 }).limit(10)

// Find queries without index (COLLSCAN)
db.system.profile.find({ 
  "planSummary": "COLLSCAN" 
}).sort({ ts: -1 })

// Find queries scanning too many documents
db.system.profile.find({
  "docsExamined": { $gt: 10000 },
  "nreturned": { $lt: 100 }
}).sort({ ts: -1 })

// Find slow queries on a specific collection
db.system.profile.find({
  ns: "mydb.orders",
  millis: { $gt: 500 }
}).sort({ millis: -1 })
```

---

## 📋 Index Management — Day-to-Day Operations

```javascript
// ═══════════════════════════════════════════
// LIST indexes
// ═══════════════════════════════════════════
db.orders.getIndexes()
// Returns array of all indexes with keys, names, options

// ═══════════════════════════════════════════
// DROP indexes
// ═══════════════════════════════════════════
db.orders.dropIndex("idx_status_1")          // Drop by name
db.orders.dropIndex({ status: 1 })            // Drop by key pattern
db.orders.dropIndexes()                       // Drop ALL indexes (except _id)!

// ═══════════════════════════════════════════
// INDEX SIZE & STATS
// ═══════════════════════════════════════════
db.orders.stats().indexSizes
// { "_id_": 856064, "idx_customer_1": 425984, ... }

db.orders.totalIndexSize()  // Total bytes used by all indexes

// ═══════════════════════════════════════════
// REBUILD indexes (rarely needed)
// ═══════════════════════════════════════════
db.orders.reIndex()  // ⚠️ Blocks the collection — use in maintenance windows

// ═══════════════════════════════════════════
// HIDE index (MongoDB 4.4+) — test impact without dropping!
// ═══════════════════════════════════════════
db.orders.hideIndex("idx_status_1")    // Query planner ignores it
// Test your queries... is performance OK without it?
db.orders.unhideIndex("idx_status_1")  // Bring it back if needed
// Way safer than dropping and recreating!
```

---

## 🧮 Index Selection Strategy — How MongoDB Chooses

```
    When multiple indexes could serve a query, MongoDB runs a "race":

    ┌──────────────────────────────────────────────────────┐
    │              Index Selection Race                     │
    │                                                      │
    │  Query: db.orders.find({status:"shipped", amt:500})  │
    │                                                      │
    │  Candidate Indexes:                                  │
    │    Plan A: { status: 1 }                             │
    │    Plan B: { amt: 1 }                                │
    │    Plan C: { status: 1, amt: 1 }                     │
    │                                                      │
    │  MongoDB races them (tries each for a short time):   │
    │                                                      │
    │    Plan A: examined 1000 keys → returned 200 docs    │
    │    Plan B: examined 1000 keys → returned 50 docs     │
    │    Plan C: examined 200 keys  → returned 200 docs 🏆 │
    │                                                      │
    │  Winner: Plan C (most efficient key-to-result ratio) │
    │                                                      │
    │  The winning plan is CACHED for similar queries.     │
    │  Cache is cleared on: index changes, server restart, │
    │  or after a threshold of writes.                     │
    └──────────────────────────────────────────────────────┘
```

---

## 🚨 Common Indexing Anti-Patterns

```
    ┌────────────────────────────────────────────────────────────────┐
    │  ❌ Anti-Pattern 1: Too Many Indexes                          │
    │     • Each index slows down writes (INSERT, UPDATE, DELETE)   │
    │     • Each index uses RAM                                     │
    │     • Rule: Keep indexes < working set of queries             │
    │     • Audit unused indexes regularly!                         │
    ├────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 2: Indexing Low-Cardinality Fields Alone     │
    │     • Index on { status: 1 } where status is "active/deleted"│
    │     • Only 2 values → index scans 50% of collection anyway   │
    │     • Fix: Make it part of a compound index                   │
    ├────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 3: Regex with Leading Wildcard               │
    │     • db.users.find({ name: /.*smith/i })                    │
    │     • CAN'T use index (must scan everything)                 │
    │     • Fix: Use /^smith/i (anchored to start) → uses index    │
    ├────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 4: $ne and $nin with Index                   │
    │     • db.orders.find({ status: { $ne: "cancelled" } })       │
    │     • Negation queries are VERY inefficient with indexes     │
    │     • Fix: Query for what you WANT, not what you don't want  │
    ├────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 5: Indexes Larger Than RAM                   │
    │     • If indexes don't fit in RAM → constant disk reads      │
    │     • Monitor: db.collection.totalIndexSize()                │
    │     • Fix: Partial indexes, TTL, archive old data            │
    ├────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 6: Not Using explain()                       │
    │     • "It's fast enough" → until it's not                    │
    │     • ALWAYS verify with explain("executionStats")           │
    │     • Especially for new queries or schema changes           │
    └────────────────────────────────────────────────────────────────┘
```

---

## 💡 Pro Tips — Index Mastery Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION INDEX CHECKLIST                        │
│                                                                     │
│  □ Every query has a supporting index (no COLLSCAN in production)   │
│  □ Compound indexes follow ESR rule (Equality → Sort → Range)      │
│  □ explain("executionStats") verified for critical queries          │
│  □ Covered queries used where possible (_id: 0 in projection)      │
│  □ Total index size fits in RAM (db.collection.totalIndexSize())    │
│  □ Unused indexes identified and dropped                           │
│  □ Partial indexes used for large collections with query filters    │
│  □ TTL indexes used for auto-expiring data (sessions, logs)        │
│  □ MongoDB profiler enabled (slowms: 100) in production            │
│  □ Index builds are done in rolling fashion in replica sets         │
│  □ Wildcard indexes ONLY for truly dynamic schemas                 │
│  □ No regex with leading wildcards in indexed queries              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔗 Quick Reference — Index Types at a Glance

| Index Type | Best For | Supports Range | Supports Sort | Supports Unique | Multikey |
|------------|----------|:-:|:-:|:-:|:-:|
| **Single Field** | General purpose | ✅ | ✅ | ✅ | ✅ |
| **Compound** | Multi-field queries | ✅ | ✅ | ✅ | ⚠️ (1 array max) |
| **Multikey** | Array fields | ✅ | ✅ | ❌ | ✅ |
| **Text** | Full-text search | ❌ | Relevance | ❌ | ✅ |
| **2dsphere** | Geo (Earth) | ✅ (distance) | ✅ (distance) | ❌ | ✅ |
| **2d** | Geo (flat) | ✅ | ✅ | ❌ | ❌ |
| **Hashed** | Shard keys | ❌ | ❌ | ❌ | ❌ |
| **Wildcard** | Dynamic schemas | ✅ | ✅ | ❌ | ✅ |
| **TTL** | Auto-expiry | N/A | N/A | ❌ | ❌ |
| **Partial** | Subset indexing | ✅ | ✅ | ✅ | ✅ |

---

## 🧭 What's Next?

Now that you can make MongoDB **blazing fast**, it's time to learn **how to design your schemas** properly:

**Next Chapter → [3B.6 MongoDB Schema Design Patterns](./06-MongoDB-Schema-Design.md)** 🟡⭐🔥

> _"The best index in the world can't fix a bad schema. Schema design is the foundation — indexes are the turbocharger."_

---

[← Previous: MongoDB Aggregation Framework](./04-MongoDB-Aggregation.md) | [Index](../INDEX.md) | [Next: MongoDB Schema Design Patterns →](./06-MongoDB-Schema-Design.md)
