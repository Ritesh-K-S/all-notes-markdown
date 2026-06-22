# 🍃 Chapter 8.3 — MongoDB Interview Questions — Complete Guide

> **"MongoDB interviews aren't just about CRUD. They test your understanding of document modeling, aggregation pipelines, and distributed system trade-offs. That's where most candidates stumble."** — Staff Engineer, MongoDB Inc.

---

## 📌 Metadata

| Field | Value |
|-------|-------|
| **Level** | 🟡 Intermediate → 🔴 Advanced |
| **Time to Master** | ~4-5 hours |
| **Prerequisites** | MongoDB chapters (Part 3B), NoSQL Foundations (3A) |
| **Interview Coverage** | Backend roles, Full-stack, DevOps, Data Engineer |

---

## 🎯 What You'll Master

- ✅ 70+ MongoDB interview questions with expert-level answers
- ✅ Schema design patterns that interviewers love to ask about
- ✅ Aggregation framework deep-dive questions
- ✅ Replication, sharding, and production questions
- ✅ Comparison questions (MongoDB vs SQL, vs other NoSQL)
- ✅ Performance tuning and indexing strategies
- ✅ Security and transaction questions

---

## 📋 Question Categories

```
┌─────────────────────────────────────────────────────────────────────┐
│             MONGODB INTERVIEW — QUESTION MAP                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SECTION 1: Basics & Core Concepts           (Q1 – Q15)           │
│  SECTION 2: Schema Design & Data Modeling    (Q16 – Q30)          │
│  SECTION 3: Aggregation Framework            (Q31 – Q40)          │
│  SECTION 4: Indexing & Performance           (Q41 – Q50)          │
│  SECTION 5: Replication & Sharding           (Q51 – Q60)          │
│  SECTION 6: Transactions, Security & Ops     (Q61 – Q70)          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

# 🟢 SECTION 1: Basics & Core Concepts (Q1–Q15)

---

### Q1: What is MongoDB? How is it different from a relational database? ⭐

```
┌──────────────────┬─────────────────────┬─────────────────────────┐
│ Feature          │ MongoDB             │ Relational (SQL)        │
├──────────────────┼─────────────────────┼─────────────────────────┤
│ Data model       │ Document (JSON/BSON)│ Tables (rows & columns) │
│ Schema           │ Flexible (dynamic)  │ Fixed (predefined)      │
│ Query language   │ MQL (MongoDB Query) │ SQL                     │
│ JOINs            │ $lookup (limited)   │ Full JOIN support       │
│ Transactions     │ Multi-doc ACID (4.0+)│ Full ACID              │
│ Scaling          │ Horizontal (sharding)│ Mostly vertical        │
│ Storage          │ BSON documents      │ Row-oriented pages      │
│ Relationships    │ Embedded + Reference│ Foreign Keys            │
│ Terminology      │ Collection          │ Table                   │
│                  │ Document            │ Row                     │
│                  │ Field               │ Column                  │
└──────────────────┴─────────────────────┴─────────────────────────┘
```

---

### Q2: What is BSON? Why does MongoDB use it instead of JSON?

```
JSON:                              BSON (Binary JSON):
┌──────────────────────┐          ┌──────────────────────┐
│ Human-readable text  │          │ Binary encoding      │
│ Limited types:       │          │ Rich types:          │
│   String, Number,    │          │   String, Int32,     │
│   Boolean, Array,    │          │   Int64, Double,     │
│   Object, null       │          │   Decimal128, Date,  │
│                      │          │   ObjectId, Binary,  │
│ No Date type         │          │   Regex, Timestamp   │
│ No Binary type       │          │                      │
│ No Integer vs Float  │          │ Traversable (skip    │
│                      │          │  to specific field)  │
│ Must parse entire    │          │ Length-prefixed →     │
│ string to find field │          │  fast field access    │
└──────────────────────┘          └──────────────────────┘

BSON Advantages:
  ✅ More data types (dates, binary, decimal128 for financials)
  ✅ Faster to encode/decode
  ✅ Traversable without parsing entire document
  ✅ Length-prefixed (know field size without reading all bytes)
```

---

### Q3: What is an ObjectId? How is it structured?

```
ObjectId: A 12-byte unique identifier for every document.

  ObjectId("507f1f77bcf86cd799439011")
  
  ┌──────────┬──────┬────────┬───────────┐
  │ 4 bytes  │ 5 bytes      │ 3 bytes   │
  │ Timestamp│ Random Value │ Counter   │
  │(seconds) │(process-unique)│(incrementing)│
  └──────────┴──────────────┴───────────┘
  
  • Timestamp: When the document was created (extractable!)
  • Random: Machine + process identifier
  • Counter: Auto-incrementing within same second
  
  db.collection.findOne()._id.getTimestamp()
  // → ISODate("2012-10-17T20:46:22Z") — creation time from ObjectId!

WHY NOT AUTO-INCREMENT?
  • No central counter needed (works in distributed systems)
  • Generated client-side (no server round-trip)
  • Roughly time-sorted (useful for default ordering)
  • Globally unique without coordination
```

---

### Q4: What are the different data types in MongoDB?

```
┌──────────────────┬──────────────────────────────────────┐
│ Type             │ Example                              │
├──────────────────┼──────────────────────────────────────┤
│ String           │ "Hello World"                        │
│ Integer (32-bit) │ NumberInt(42)                         │
│ Long (64-bit)    │ NumberLong(9999999999)                │
│ Double           │ 3.14                                 │
│ Decimal128       │ NumberDecimal("19.99") ← financials! │
│ Boolean          │ true / false                         │
│ Date             │ ISODate("2024-01-15T10:30:00Z")     │
│ Timestamp        │ Timestamp(1234567890, 1) ← internal │
│ ObjectId         │ ObjectId("507f1f77bcf86cd799439011")  │
│ Array            │ [1, 2, 3] or ["a", "b"]             │
│ Object           │ { nested: { key: "value" } }        │
│ Binary Data      │ BinData(0, "base64...") ← images    │
│ Null             │ null                                 │
│ Regular Expr.    │ /pattern/i                           │
│ Min/Max Key      │ MinKey / MaxKey ← internal sorting  │
│ UUID             │ UUID("...") ← v4.0+                 │
└──────────────────┴──────────────────────────────────────┘
```

> ⚠️ **Gotcha**: Use `Decimal128` for financial data, NOT `Double`! `0.1 + 0.2 = 0.30000000000000004` in floating point.

---

### Q5: What is the difference between `find()` and `findOne()`?

```javascript
// findOne(): Returns a SINGLE document (or null)
db.users.findOne({ email: "alice@example.com" })
// → { _id: ObjectId("..."), name: "Alice", email: "alice@example.com" }

// find(): Returns a CURSOR to iterate over multiple documents
db.users.find({ age: { $gte: 25 } })
// → Cursor (lazy — fetches in batches of 101, then 16MB batches)

// find() with chaining
db.users.find({ age: { $gte: 25 } })
    .sort({ age: 1 })
    .limit(10)
    .skip(20)
    .projection({ name: 1, age: 1, _id: 0 })
```

---

### Q6: Explain MongoDB query operators.

```javascript
// COMPARISON
{ age: { $eq: 25 } }     // Equal (same as { age: 25 })
{ age: { $ne: 25 } }     // Not equal
{ age: { $gt: 25 } }     // Greater than
{ age: { $gte: 25 } }    // Greater than or equal
{ age: { $lt: 25 } }     // Less than
{ age: { $lte: 25 } }    // Less than or equal
{ age: { $in: [25, 30] } } // In array
{ age: { $nin: [25, 30] } }// Not in array

// LOGICAL
{ $and: [{ age: { $gte: 25 } }, { city: "NYC" }] }
{ $or: [{ age: 25 }, { age: 30 }] }
{ $not: { age: { $gt: 25 } } }
{ $nor: [{ age: 25 }, { city: "NYC" }] }

// ELEMENT
{ phone: { $exists: true } }    // Field exists
{ age: { $type: "number" } }    // Field type check

// ARRAY
{ tags: { $all: ["mongo", "db"] } }    // Array contains all
{ tags: { $size: 3 } }                  // Array has exactly 3 elements
{ scores: { $elemMatch: { $gt: 80, $lt: 90 } } } // Element matches all

// TEXT
{ $text: { $search: "coffee shop" } }   // Full-text search (needs text index)

// REGEX
{ name: { $regex: /^Ali/i } }           // Pattern matching
```

---

### Q7: What is the difference between `update()`, `updateOne()`, and `updateMany()`?

```javascript
// updateOne(): Updates FIRST matching document
db.users.updateOne(
    { email: "alice@example.com" },
    { $set: { age: 30 } }
)

// updateMany(): Updates ALL matching documents
db.users.updateMany(
    { status: "inactive" },
    { $set: { archived: true } }
)

// Update operators
{ $set: { field: value } }          // Set field value
{ $unset: { field: "" } }           // Remove field
{ $inc: { count: 1 } }             // Increment by 1
{ $mul: { price: 1.1 } }           // Multiply by 1.1
{ $rename: { old: "new" } }        // Rename field
{ $min: { low: 5 } }               // Set if less than current
{ $max: { high: 100 } }            // Set if greater than current
{ $push: { tags: "new" } }         // Add to array
{ $pull: { tags: "old" } }         // Remove from array
{ $addToSet: { tags: "unique" } }  // Add to array (only if not exists)
{ $pop: { arr: 1 } }               // Remove last (-1 for first)

// replaceOne(): Replaces ENTIRE document (except _id)
db.users.replaceOne(
    { _id: ObjectId("...") },
    { name: "Alice", age: 31, email: "alice@new.com" }
)
```

---

### Q8: What is the difference between `remove()`, `deleteOne()`, and `deleteMany()`?

```javascript
// deleteOne(): Delete FIRST matching document
db.users.deleteOne({ status: "deleted" })

// deleteMany(): Delete ALL matching documents
db.users.deleteMany({ lastLogin: { $lt: ISODate("2023-01-01") } })

// deleteMany({}): Delete ALL documents (⚠️ dangerous! keeps collection)
db.users.deleteMany({})

// drop(): Delete entire COLLECTION (structure + data + indexes)
db.users.drop()
```

> ⚠️ **`remove()` is deprecated in modern drivers.** Use `deleteOne()` / `deleteMany()`.

---

### Q9: What is a Capped Collection?

```javascript
// Capped Collection: Fixed-size, auto-deletes oldest documents
db.createCollection("logs", {
    capped: true,
    size: 1048576,    // Max size in bytes (1MB)
    max: 1000         // Max number of documents
})

// Properties:
// ✅ Insertion order guaranteed
// ✅ Oldest documents auto-deleted when full (circular buffer)
// ✅ Very fast writes (no index overhead)
// ❌ Cannot delete individual documents
// ❌ Cannot shard capped collections
// ❌ Cannot change size after creation

// USE CASES:
// • Application logs (last N events)
// • Cache (auto-eviction)
// • Circular buffers (sensor data)
// • Tailable cursors (real-time monitoring, like tail -f)
```

---

### Q10: What is the WiredTiger storage engine?

```
WIREDTIGER: Default storage engine since MongoDB 3.2

┌────────────────────────────────────────────────────────────┐
│                   WiredTiger Features                       │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  📦 COMPRESSION:                                           │
│  • snappy (default), zlib, zstd                           │
│  • 50-80% storage reduction                               │
│                                                            │
│  🔒 CONCURRENCY:                                           │
│  • Document-level locking (not collection-level!)         │
│  • MVCC for reads (no read locks)                         │
│                                                            │
│  💾 CACHING:                                               │
│  • Internal cache (default: 50% RAM - 1GB, or 256MB min) │
│  • Configurable: --wiredTigerCacheSizeGB                  │
│                                                            │
│  📝 JOURNALING:                                            │
│  • Write-Ahead Log (WAL) for durability                   │
│  • Checkpoints every 60 seconds                           │
│  • Journal synced to disk on commit                       │
│                                                            │
│  🗂️ STORAGE:                                               │
│  • B-Tree for indexes                                     │
│  • Separate files per collection and index                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

### Q11: What is the `$lookup` operator? Is it the same as SQL JOIN?

```javascript
// $lookup: Left Outer Join in aggregation pipeline
db.orders.aggregate([
    {
        $lookup: {
            from: "customers",           // "right" collection
            localField: "customer_id",   // field in orders
            foreignField: "_id",         // field in customers
            as: "customer_info"          // output array field
        }
    },
    { $unwind: "$customer_info" }  // Flatten the array to object
])
```

```
$lookup vs SQL JOIN:
┌────────────────────┬──────────────────┬──────────────────────┐
│ Feature            │ $lookup          │ SQL JOIN             │
├────────────────────┼──────────────────┼──────────────────────┤
│ Type               │ Left Outer only* │ Inner, Left, Right,  │
│                    │                  │ Full, Cross          │
│ Performance        │ 🐌 Slower        │ ⚡ Optimized          │
│ Sharded colls      │ ⚠️ "from" can be│ ✅ No restrictions   │
│                    │  sharded (5.1+)  │                      │
│ Use frequency      │ Discouraged      │ Encouraged           │
│ Alternative        │ Embed data!      │ N/A                  │
│ Indexes            │ Use on foreignField│ Use on JOIN columns│
└────────────────────┴──────────────────┴──────────────────────┘
* Subpipeline $lookup can simulate other join types
```

> 💡 **Pro Tip**: If you find yourself using `$lookup` frequently, your schema design may be wrong. MongoDB favors **embedding** over **referencing**.

---

### Q12: What are MongoDB Projections?

```javascript
// PROJECTION: Control which fields are returned

// Include specific fields (inclusion)
db.users.find({}, { name: 1, email: 1 })
// → { _id: ObjectId("..."), name: "Alice", email: "alice@..." }
// Note: _id is included by default!

// Exclude specific fields (exclusion)
db.users.find({}, { password: 0, ssn: 0 })
// → Everything EXCEPT password and ssn

// Exclude _id explicitly
db.users.find({}, { name: 1, email: 1, _id: 0 })
// → { name: "Alice", email: "alice@..." }

// ⚠️ CANNOT MIX inclusion and exclusion (except _id)
db.users.find({}, { name: 1, password: 0 })  // ❌ ERROR!
```

---

### Q13: What is the difference between `$push` and `$addToSet`?

```javascript
// Starting document: { tags: ["mongodb", "database"] }

// $push: Always adds (even if duplicate)
db.posts.updateOne({}, { $push: { tags: "mongodb" } })
// → { tags: ["mongodb", "database", "mongodb"] }  ← duplicate!

// $addToSet: Only adds if not already present
db.posts.updateOne({}, { $addToSet: { tags: "mongodb" } })
// → { tags: ["mongodb", "database"] }  ← no change (already exists)

db.posts.updateOne({}, { $addToSet: { tags: "nosql" } })
// → { tags: ["mongodb", "database", "nosql"] }  ← added (new!)

// $push with $each: Add multiple
db.posts.updateOne({}, { $push: { tags: { $each: ["a", "b", "c"] } } })

// $push with $sort and $slice: Maintain sorted, capped array
db.users.updateOne({}, { 
    $push: { 
        scores: { 
            $each: [85], 
            $sort: -1,      // Sort descending
            $slice: 5        // Keep only top 5
        } 
    } 
})
```

---

### Q14: What is `explain()` in MongoDB?

```javascript
// explain() shows query execution plan

// Verbosity levels:
db.users.find({ age: 25 }).explain("queryPlanner")    // Default: show plan
db.users.find({ age: 25 }).explain("executionStats")   // Show actual stats
db.users.find({ age: 25 }).explain("allPlansExecution") // Show all candidates

// KEY THINGS TO CHECK:
// {
//   "winningPlan": {
//     "stage": "IXSCAN",          ← ✅ Using index
//     "indexName": "age_1",
//     "direction": "forward"
//   },
//   "executionStats": {
//     "nReturned": 5,             ← Documents returned
//     "totalDocsExamined": 5,     ← Documents scanned
//     "totalKeysExamined": 5,     ← Index keys scanned
//     "executionTimeMillis": 0    ← Time in ms
//   }
// }

// RED FLAGS:
// "stage": "COLLSCAN"            ← 🔴 Full collection scan!
// totalDocsExamined >> nReturned  ← 🔴 Scanning too many docs
// "stage": "SORT"                 ← 🟡 In-memory sort (needs index)
```

---

### Q15: What is MongoDB Atlas?

```
MONGODB ATLAS: Fully managed MongoDB cloud service.

┌────────────────────────────────────────────────────────────┐
│                    MongoDB Atlas                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ☁️ DEPLOYMENT:                                             │
│  • AWS, Azure, GCP                                        │
│  • Free tier (M0: 512MB)                                  │
│  • Serverless instances (pay-per-operation)                │
│                                                            │
│  🔍 FEATURES:                                               │
│  • Atlas Search (Lucene-based full-text search)           │
│  • Atlas Data Lake (query S3/Azure Blob)                  │
│  • Atlas Charts (built-in visualization)                  │
│  • Atlas Device Sync (mobile offline-first)               │
│  • Atlas Triggers (serverless functions)                   │
│  • Atlas Vector Search (AI/ML embeddings)                  │
│                                                            │
│  🔒 SECURITY:                                               │
│  • Network peering / Private endpoints                    │
│  • Encryption at rest + in transit                        │
│  • Field-level encryption (client-side)                   │
│  • LDAP, SCRAM, x.509 authentication                     │
│                                                            │
│  📊 MONITORING:                                             │
│  • Real-Time Performance Panel                            │
│  • Query Profiler                                         │
│  • Automated backups with PITR                            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

# 🏗️ SECTION 2: Schema Design & Data Modeling (Q16–Q30)

---

### Q16: Embedding vs Referencing — When to use which? ⭐🔥

```
EMBEDDING (Denormalized):               REFERENCING (Normalized):
┌────────────────────────┐              ┌──────────────┐  ┌──────────┐
│ {                      │              │ {            │  │ {        │
│   _id: 1,              │              │   _id: 1,    │  │  _id: 101│
│   name: "Alice",       │              │   name: "Ali"│  │  street: │
│   address: {           │              │   address_id:│  │  "123 St"│
│     street: "123 St",  │              │     101      │  │ }        │
│     city: "NYC"        │              │ }            │  └──────────┘
│   }                    │              └──────────────┘
│ }                      │
└────────────────────────┘

EMBED WHEN:                           REFERENCE WHEN:
✅ 1:1 relationship                   ✅ Many-to-many relationship
✅ 1:few (author → few books)        ✅ 1:many (large, unbounded)
✅ Data read together                 ✅ Data accessed independently
✅ Data rarely changes               ✅ Data changes frequently
✅ Document stays under 16MB         ✅ Document would exceed 16MB
✅ Atomic operations needed          ✅ Multiple collections share data
```

**🎯 The Golden Rule**: "Data that is accessed together should be stored together."

---

### Q17: What are the MongoDB Schema Design Patterns?

```
┌──────────────────────┬──────────────────────────────────────────┐
│ Pattern              │ Use Case                                 │
├──────────────────────┼──────────────────────────────────────────┤
│ Attribute            │ Many similar fields → key-value array    │
│                      │ { specs: [{k:"color", v:"red"}, ...] }  │
├──────────────────────┼──────────────────────────────────────────┤
│ Bucket               │ Time-series data → group into buckets   │
│                      │ { date: "2024-01", readings: [...] }    │
├──────────────────────┼──────────────────────────────────────────┤
│ Computed              │ Pre-compute expensive calculations      │
│                      │ { total_views: 15000 } ← updated on write│
├──────────────────────┼──────────────────────────────────────────┤
│ Outlier              │ Handle documents that break the pattern  │
│                      │ Normal: embed. Outlier: overflow doc     │
├──────────────────────┼──────────────────────────────────────────┤
│ Subset               │ Store frequently-accessed subset in doc  │
│                      │ { recent_reviews: [last 10] }           │
├──────────────────────┼──────────────────────────────────────────┤
│ Extended Reference   │ Copy frequently-accessed fields from     │
│                      │ referenced doc to avoid $lookup          │
├──────────────────────┼──────────────────────────────────────────┤
│ Polymorphic          │ Documents with different structures in   │
│                      │ same collection (products: books, phones)│
├──────────────────────┼──────────────────────────────────────────┤
│ Schema Versioning    │ { schema_version: 2, ... } for migrations│
└──────────────────────┴──────────────────────────────────────────┘
```

---

### Q18: How would you model a one-to-many relationship?

```javascript
// OPTION 1: Embed (1:few → author has few addresses)
{
    _id: ObjectId("auth1"),
    name: "Alice",
    addresses: [
        { type: "home", city: "New York" },
        { type: "work", city: "San Francisco" }
    ]
}
// ✅ One read fetches everything
// ❌ Document can grow unbounded if "few" becomes "many"

// OPTION 2: Reference (1:many → author has many books)
// Authors collection
{ _id: ObjectId("auth1"), name: "Alice" }

// Books collection
{ _id: ObjectId("book1"), title: "MongoDB Guide", author_id: ObjectId("auth1") }
{ _id: ObjectId("book2"), title: "NoSQL Patterns", author_id: ObjectId("auth1") }
// ✅ Scales to unlimited books
// ❌ Requires $lookup or multiple queries

// OPTION 3: Hybrid (1:many → order has many items, but items are part of order)
{
    _id: ObjectId("order1"),
    customer_id: ObjectId("cust1"),
    customer_name: "Alice",             // ← Extended reference (denormalized)
    items: [
        { product_id: ObjectId("p1"), name: "Laptop", qty: 1, price: 999 },
        { product_id: ObjectId("p2"), name: "Mouse", qty: 2, price: 25 }
    ],
    total: 1049
}
```

---

### Q19: How would you model a many-to-many relationship?

```javascript
// SQL approach: Junction table
// MongoDB approach: Array of references (in one or both sides)

// OPTION 1: Array of references in one collection
// Students collection
{
    _id: ObjectId("s1"),
    name: "Alice",
    course_ids: [ObjectId("c1"), ObjectId("c2"), ObjectId("c3")]
}

// Courses collection
{
    _id: ObjectId("c1"),
    name: "Database Systems",
    student_ids: [ObjectId("s1"), ObjectId("s2")]  // Optional: bidirectional
}

// OPTION 2: Intermediate collection (for extra data on relationship)
// Enrollments collection
{
    student_id: ObjectId("s1"),
    course_id: ObjectId("c1"),
    enrolled_date: ISODate("2024-01-15"),
    grade: "A",
    semester: "Spring 2024"
}
```

---

### Q20: What is the 16MB document size limit? How do you handle it?

```
MongoDB maximum document size: 16 MB (BSON)

WHY 16MB?
  • Prevents single documents from consuming too much memory
  • Network transfer efficiency
  • Keeps working set manageable

WHAT TO DO IF YOU HIT IT:
  1. RESTRUCTURE: Break into multiple documents
     Instead of: { comments: [10,000 comments] }
     Do: Separate comments collection with post_id reference
     
  2. BUCKET PATTERN: Group into chunks
     Instead of: 1 doc with 1M sensor readings
     Do: 1 doc per hour with ~3,600 readings each
     
  3. GRIDFS: For files > 16MB
     • Splits file into 255KB chunks
     • Stores chunks in fs.chunks collection
     • Metadata in fs.files collection
     • Built-in with MongoDB drivers

  4. SUBSET PATTERN: Store only recent/relevant subset
     • Keep last 10 reviews in document
     • Full history in separate collection
```

---

### Q21: What is GridFS?

```
GRIDFS: MongoDB's specification for storing files larger than 16MB.

  ┌──────────────┐         ┌─────────────────────┐
  │  Large File  │         │  fs.files            │
  │  (50MB PDF)  │ ────→   │  { filename, size,   │
  └──────────────┘         │    uploadDate, md5 }  │
        │                  └─────────────────────┘
        │ Split into                 │
        │ 255KB chunks               │ references
        ↓                            ↓
  ┌───────────────────────────────────────┐
  │  fs.chunks                            │
  │  { files_id, n: 0, data: Binary }    │
  │  { files_id, n: 1, data: Binary }    │
  │  { files_id, n: 2, data: Binary }    │
  │  ...                                  │
  └───────────────────────────────────────┘
```

```javascript
// Using GridFS with Node.js
const bucket = new GridFSBucket(db);
fs.createReadStream('./large-file.pdf')
    .pipe(bucket.openUploadStream('large-file.pdf'));

// Downloading
bucket.openDownloadStreamByName('large-file.pdf')
    .pipe(fs.createWriteStream('./downloaded.pdf'));
```

---

### Q22: How do you handle schema migrations in MongoDB?

```javascript
// STRATEGY 1: Schema Versioning Pattern
{
    _id: ObjectId("..."),
    schema_version: 2,
    name: "Alice",
    email: "alice@example.com",
    address: { street: "123 St", city: "NYC" }  // Added in v2
}

// Application code handles multiple versions:
if (doc.schema_version === 1) {
    // Old format: address was a string
    doc.address = { street: doc.address, city: "Unknown" };
    doc.schema_version = 2;
}

// STRATEGY 2: Lazy Migration (update on read/write)
// When a document is accessed, update to new schema

// STRATEGY 3: Bulk Migration Script
db.users.updateMany(
    { schema_version: { $lt: 2 } },
    [
        { $set: { 
            address: { street: "$address_line", city: "$city" },
            schema_version: 2 
        }},
        { $unset: ["address_line", "city"] }
    ]
);
```

---

### Q23: What is the Anti-Pattern of massive arrays?

```
❌ ANTI-PATTERN: Unbounded array growth

// BAD: Social media post with ALL comments embedded
{
    _id: ObjectId("post1"),
    title: "My Post",
    comments: [
        // This can grow to 100,000+ comments!
        // → Document exceeds 16MB 💀
        // → Working set too large
        // → Array operations ($push) become slow
    ]
}

✅ SOLUTION: Separate collection with reference
// Posts collection
{ _id: ObjectId("post1"), title: "My Post", comment_count: 1523 }

// Comments collection  
{ _id: ObjectId("c1"), post_id: ObjectId("post1"), author: "Bob", text: "..." }
{ _id: ObjectId("c2"), post_id: ObjectId("post1"), author: "Eve", text: "..." }

RULES OF THUMB:
  Embed: Array will have < 100 items (addresses, phone numbers)
  Reference: Array could grow beyond 100 items (comments, likes, orders)
  Hybrid: Embed last N items + reference full collection
```

---

### Q24: How do you design a MongoDB schema for an e-commerce platform?

```javascript
// PRODUCTS (polymorphic — different product types)
{
    _id: ObjectId("p1"),
    name: "MacBook Pro",
    category: "electronics",
    price: NumberDecimal("1999.99"),
    stock: 150,
    specs: [
        { key: "ram", value: "16GB", unit: "GB" },
        { key: "storage", value: "512", unit: "GB" }
    ],
    reviews_summary: {
        avg_rating: 4.5,
        count: 1234,
        recent: [/* last 5 reviews embedded */]
    },
    images: ["url1", "url2"]
}

// ORDERS (embedded items — accessed together, immutable after creation)
{
    _id: ObjectId("ord1"),
    customer: {  // Extended reference (snapshot at order time)
        _id: ObjectId("c1"),
        name: "Alice",
        email: "alice@example.com"
    },
    items: [
        {
            product_id: ObjectId("p1"),
            name: "MacBook Pro",     // Snapshot! Don't lookup at read time
            price: NumberDecimal("1999.99"),
            quantity: 1
        }
    ],
    total: NumberDecimal("1999.99"),
    status: "shipped",
    shipping_address: { /* embedded */ },
    created_at: ISODate("2024-01-15"),
    updated_at: ISODate("2024-01-16")
}

// USERS
{
    _id: ObjectId("c1"),
    name: "Alice",
    email: "alice@example.com",
    addresses: [/* embedded — few per user */],
    payment_methods: [/* embedded — few per user */]
}
```

---

### Q25: What is the Subset Pattern?

```javascript
// PROBLEM: Product has 10,000 reviews. Loading all = slow.
// SOLUTION: Embed a SUBSET + keep full set in separate collection.

// Products collection (with last 10 reviews embedded)
{
    _id: ObjectId("p1"),
    name: "Widget",
    reviews_summary: { avg: 4.2, count: 10000 },
    recent_reviews: [
        { author: "Alice", rating: 5, text: "Great!", date: ISODate("...") },
        { author: "Bob", rating: 4, text: "Good", date: ISODate("...") },
        // ... last 10 only
    ]
}

// Reviews collection (ALL reviews)
{
    product_id: ObjectId("p1"),
    author: "Alice",
    rating: 5,
    text: "Great!",
    date: ISODate("...")
}

// BENEFIT: Product page loads fast (embedded reviews)
// TRADE-OFF: When user clicks "See all reviews" → query reviews collection
```

---

### Q26-Q30: Quick Schema Design Questions

**Q26: How do you model a tree/hierarchy in MongoDB?**
```javascript
// OPTION 1: Parent Reference (simplest)
{ _id: "CEO", parent: null }
{ _id: "VP_Eng", parent: "CEO" }
{ _id: "Dev_Lead", parent: "VP_Eng" }

// OPTION 2: Array of Ancestors (fast ancestor queries)
{ _id: "Dev_Lead", ancestors: ["CEO", "VP_Eng"], parent: "VP_Eng" }

// OPTION 3: Materialized Path
{ _id: "Dev_Lead", path: "/CEO/VP_Eng/Dev_Lead" }
// Find all descendants: db.org.find({ path: /^\/CEO\/VP_Eng/ })
```

**Q27: What is the Polymorphic Pattern?**
```javascript
// Same collection, different shapes
{ type: "book", title: "...", pages: 300, isbn: "..." }
{ type: "movie", title: "...", duration: 120, director: "..." }
{ type: "song", title: "...", duration: 210, artist: "..." }
// All queryable with: db.media.find({ title: /search/ })
```

**Q28: What is the Bucket Pattern?**
```javascript
// Instead of 1 doc per sensor reading (millions of docs):
// Group readings into hourly buckets
{
    sensor_id: "temp_001",
    date: ISODate("2024-01-15T10:00:00Z"),
    readings: [
        { time: ISODate("...T10:00:05Z"), value: 22.5 },
        { time: ISODate("...T10:00:10Z"), value: 22.7 },
        // ... 720 readings per hour (5-sec intervals)
    ],
    count: 720,
    sum: 16200,      // Pre-computed for fast AVG
    min: 21.8,
    max: 23.1
}
```

**Q29: Should you use MongoDB for financial transactions?**
```
Since MongoDB 4.0+ supports multi-document ACID transactions: YES, it's possible.
BUT: SQL databases are still preferred for financial systems because:
  • Decades of battle-tested ACID compliance
  • Stronger isolation levels
  • Better tooling for financial reporting (SQL JOINs)
  • Regulatory compliance easier to demonstrate
  
MongoDB is fine for: Fintech apps, payment metadata, audit logs
Not ideal for: Core banking ledger, stock exchange matching engine
```

**Q30: What are Read Concerns and Write Concerns?**
```javascript
// WRITE CONCERN: How many nodes must acknowledge a write
db.orders.insertOne(
    { item: "widget" },
    { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
// w: 1       → Primary only (fast, risk of data loss)
// w: majority → Majority of replica set (safe)
// j: true    → Write to journal (durable)
// w: 0       → Fire and forget (fastest, no ack)

// READ CONCERN: What data is visible to reads
// "local"      → Latest data on this node (default, may be rolled back)
// "majority"   → Data acknowledged by majority (safe, slightly stale)
// "linearizable"→ Most recent committed data (slowest, strongest)
// "snapshot"   → Consistent snapshot (for transactions)
```

---

# ⚡ SECTION 3: Aggregation Framework (Q31–Q40)

---

### Q31: What is the Aggregation Framework? How is it different from find()? ⭐

```javascript
// find(): Simple filtering and projection
db.orders.find({ status: "shipped" }, { customer: 1, total: 1 })

// aggregate(): Multi-stage pipeline for complex transformations
db.orders.aggregate([
    { $match: { status: "shipped" } },                    // Filter
    { $group: { _id: "$customer", total: { $sum: "$amount" } } }, // Group
    { $sort: { total: -1 } },                             // Sort
    { $limit: 10 }                                        // Top 10
])
```

```
AGGREGATION PIPELINE:
  Documents → [$match] → [$group] → [$sort] → [$limit] → Results
                 ↓           ↓          ↓          ↓
              Filter      Transform   Order      Trim
              (WHERE)     (GROUP BY)  (ORDER BY) (LIMIT)
```

---

### Q32: List the most important aggregation stages.

```
┌──────────────────┬──────────────────────────────────────────────┐
│ Stage            │ Purpose (SQL Equivalent)                     │
├──────────────────┼──────────────────────────────────────────────┤
│ $match           │ Filter documents (WHERE)                     │
│ $group           │ Group and aggregate (GROUP BY)               │
│ $project         │ Reshape/select fields (SELECT)               │
│ $sort            │ Order results (ORDER BY)                     │
│ $limit           │ Limit results (LIMIT)                        │
│ $skip            │ Skip results (OFFSET)                        │
│ $unwind          │ Deconstruct array (unnest)                   │
│ $lookup          │ Join collections (LEFT JOIN)                 │
│ $addFields       │ Add computed fields                          │
│ $set             │ Alias for $addFields                         │
│ $count           │ Count documents (COUNT(*))                   │
│ $facet           │ Multiple pipelines in parallel               │
│ $bucket          │ Group into ranges (histogram)                │
│ $merge           │ Write results to collection                  │
│ $out             │ Replace collection with results              │
│ $replaceRoot     │ Replace document with subdocument            │
│ $graphLookup     │ Recursive lookup (hierarchies)               │
│ $unionWith       │ Combine from multiple collections (UNION)    │
│ $densify         │ Fill gaps in time-series                     │
│ $setWindowFields │ Window functions (ROW_NUMBER, RANK, etc.)    │
└──────────────────┴──────────────────────────────────────────────┘
```

---

### Q33: Write an aggregation to find total revenue per customer, sorted descending. ⭐

```javascript
db.orders.aggregate([
    { $match: { status: { $ne: "cancelled" } } },
    { $group: {
        _id: "$customer_id",
        total_revenue: { $sum: "$amount" },
        order_count: { $sum: 1 },
        avg_order: { $avg: "$amount" },
        last_order: { $max: "$order_date" }
    }},
    { $sort: { total_revenue: -1 } },
    { $limit: 10 },
    { $lookup: {
        from: "customers",
        localField: "_id",
        foreignField: "_id",
        as: "customer"
    }},
    { $unwind: "$customer" },
    { $project: {
        customer_name: "$customer.name",
        total_revenue: 1,
        order_count: 1,
        avg_order: { $round: ["$avg_order", 2] },
        last_order: 1
    }}
])
```

---

### Q34: What is `$unwind` and when do you need it?

```javascript
// $unwind: Deconstructs an array field → one document per element

// Before $unwind:
{ _id: 1, name: "Alice", tags: ["mongodb", "database", "nosql"] }

// After { $unwind: "$tags" }:
{ _id: 1, name: "Alice", tags: "mongodb" }
{ _id: 1, name: "Alice", tags: "database" }
{ _id: 1, name: "Alice", tags: "nosql" }

// USE CASES:
// 1. Count tag frequency across all documents
db.posts.aggregate([
    { $unwind: "$tags" },
    { $group: { _id: "$tags", count: { $sum: 1 } } },
    { $sort: { count: -1 } }
])

// 2. Before $group on array elements
// 3. After $lookup (results come as array)

// Handle empty arrays:
{ $unwind: { path: "$tags", preserveNullAndEmptyArrays: true } }
// Without this flag: Documents with no tags are DROPPED!
```

---

### Q35: How do you do pagination in aggregation?

```javascript
// METHOD 1: $skip + $limit (simple but slow for deep pages)
db.products.aggregate([
    { $match: { category: "electronics" } },
    { $sort: { price: -1 } },
    { $skip: 100 },    // Skip first 100
    { $limit: 20 }     // Return 20
])
// ⚠️ Problem: $skip scans and discards N documents → slow for page 500!

// METHOD 2: Range-based / Keyset pagination (fast!) ⚡
// First page:
db.products.aggregate([
    { $match: { category: "electronics" } },
    { $sort: { price: -1, _id: -1 } },
    { $limit: 20 }
])
// → Last doc has price: 500, _id: ObjectId("abc")

// Next page:
db.products.aggregate([
    { $match: { 
        category: "electronics",
        $or: [
            { price: { $lt: 500 } },
            { price: 500, _id: { $lt: ObjectId("abc") } }
        ]
    }},
    { $sort: { price: -1, _id: -1 } },
    { $limit: 20 }
])
```

---

### Q36: What is `$facet`? When would you use it?

```javascript
// $facet: Run MULTIPLE aggregation pipelines on the SAME input
// Perfect for: Dashboards, search results with metadata

db.products.aggregate([
    { $match: { category: "electronics" } },
    { $facet: {
        // Pipeline 1: Paginated results
        "results": [
            { $sort: { price: -1 } },
            { $skip: 0 },
            { $limit: 10 }
        ],
        // Pipeline 2: Total count
        "total": [
            { $count: "count" }
        ],
        // Pipeline 3: Price range stats
        "priceStats": [
            { $group: {
                _id: null,
                min: { $min: "$price" },
                max: { $max: "$price" },
                avg: { $avg: "$price" }
            }}
        ],
        // Pipeline 4: Brand breakdown
        "byBrand": [
            { $group: { _id: "$brand", count: { $sum: 1 } } },
            { $sort: { count: -1 } }
        ]
    }}
])
// Returns ONE document with all four results!
```

---

### Q37: How do you use Window Functions in MongoDB?

```javascript
// $setWindowFields: MongoDB 5.0+ (equivalent to SQL window functions!)

db.sales.aggregate([
    { $setWindowFields: {
        partitionBy: "$region",
        sortBy: { date: 1 },
        output: {
            // Running total per region
            runningTotal: {
                $sum: "$amount",
                window: { documents: ["unbounded", "current"] }
            },
            // Rank within region
            rank: {
                $rank: {}
            },
            // Moving average (3-day)
            movingAvg: {
                $avg: "$amount",
                window: { documents: [-2, 0] }
            },
            // Previous day's amount (LAG)
            prevDayAmount: {
                $shift: { output: "$amount", by: -1, default: 0 }
            }
        }
    }}
])
```

---

### Q38: Write an aggregation for "Top 3 products per category."

```javascript
db.products.aggregate([
    { $sort: { category: 1, sales: -1 } },
    { $group: {
        _id: "$category",
        products: { $push: {
            name: "$name",
            sales: "$sales",
            price: "$price"
        }}
    }},
    { $project: {
        category: "$_id",
        top3: { $slice: ["$products", 3] }
    }}
])

// Alternative with $setWindowFields (MongoDB 5.0+):
db.products.aggregate([
    { $setWindowFields: {
        partitionBy: "$category",
        sortBy: { sales: -1 },
        output: { rank: { $rank: {} } }
    }},
    { $match: { rank: { $lte: 3 } } },
    { $project: { rank: 0 } }
])
```

---

### Q39: What is `$graphLookup`? How is it different from `$lookup`?

```javascript
// $graphLookup: Recursive lookup for hierarchical/graph data
// Find ALL ancestors of an employee (up the org chart)

db.employees.aggregate([
    { $match: { name: "Junior Dev" } },
    { $graphLookup: {
        from: "employees",
        startWith: "$manager_id",      // Start from this field
        connectFromField: "manager_id", // Follow this field
        connectToField: "_id",          // Match against this field
        as: "management_chain",         // Output array
        maxDepth: 10,                   // Max recursion depth
        depthField: "level"             // Distance from start
    }}
])

// Result:
// {
//   name: "Junior Dev",
//   management_chain: [
//     { name: "Team Lead", level: 0 },
//     { name: "VP Engineering", level: 1 },
//     { name: "CEO", level: 2 }
//   ]
// }
```

---

### Q40: How do you optimize aggregation pipeline performance?

```
AGGREGATION OPTIMIZATION RULES:

┌─────────────────────────────────────────────────────────────┐
│ 1. $match FIRST — Filter early, reduce data flowing through│
│    pipeline. Place $match before $group, $lookup, etc.     │
│                                                            │
│ 2. $match + $sort use INDEXES                              │
│    Only at the BEGINNING of the pipeline!                  │
│    After $group or $project → no index usage.             │
│                                                            │
│ 3. $project early — Remove unneeded fields to reduce       │
│    memory usage in subsequent stages.                      │
│                                                            │
│ 4. $limit early — If you only need top 10, limit ASAP.    │
│                                                            │
│ 5. Avoid $unwind on large arrays — Creates many documents. │
│    Use $filter instead if possible.                        │
│                                                            │
│ 6. Use { allowDiskUse: true } for large datasets           │
│    Default: 100MB memory limit per stage.                  │
│                                                            │
│ 7. $merge/$out to write results to collection              │
│    → Pre-compute expensive aggregations.                   │
│                                                            │
│ 8. Check with explain():                                   │
│    db.coll.explain().aggregate([...])                      │
└─────────────────────────────────────────────────────────────┘
```

---

# 🔍 SECTION 4: Indexing & Performance (Q41–Q50)

---

### Q41: What types of indexes does MongoDB support? ⭐

```
┌──────────────────────┬──────────────────────────────────────────┐
│ Index Type           │ Use Case                                 │
├──────────────────────┼──────────────────────────────────────────┤
│ Single Field         │ db.coll.createIndex({ age: 1 })         │
│                      │ Most common. 1=asc, -1=desc             │
├──────────────────────┼──────────────────────────────────────────┤
│ Compound             │ db.coll.createIndex({ dept: 1, sal: -1})│
│                      │ Multi-field. ORDER MATTERS!              │
├──────────────────────┼──────────────────────────────────────────┤
│ Multikey             │ Automatic for array fields               │
│                      │ One index entry per array element        │
├──────────────────────┼──────────────────────────────────────────┤
│ Text                 │ db.coll.createIndex({ desc: "text" })   │
│                      │ Full-text search. One per collection!    │
├──────────────────────┼──────────────────────────────────────────┤
│ Geospatial (2dsphere)│ db.coll.createIndex({ loc: "2dsphere" })│
│                      │ Queries: $near, $geoWithin              │
├──────────────────────┼──────────────────────────────────────────┤
│ Hashed               │ db.coll.createIndex({ user_id: "hashed"})│
│                      │ For hash-based sharding. Equality only.  │
├──────────────────────┼──────────────────────────────────────────┤
│ Wildcard             │ db.coll.createIndex({ "$**": 1 })       │
│                      │ Index ALL fields. Good for dynamic schema│
├──────────────────────┼──────────────────────────────────────────┤
│ TTL (Time-To-Live)   │ db.coll.createIndex({ createdAt: 1 },  │
│                      │   { expireAfterSeconds: 3600 })         │
│                      │ Auto-delete documents after time!        │
├──────────────────────┼──────────────────────────────────────────┤
│ Unique               │ db.coll.createIndex({ email: 1 },      │
│                      │   { unique: true })                     │
├──────────────────────┼──────────────────────────────────────────┤
│ Sparse               │ db.coll.createIndex({ phone: 1 },      │
│                      │   { sparse: true })                     │
│                      │ Only index docs WHERE field EXISTS       │
├──────────────────────┼──────────────────────────────────────────┤
│ Partial              │ db.coll.createIndex({ salary: 1 },     │
│                      │   { partialFilterExpression:            │
│                      │     { status: "active" } })             │
│                      │ Only index docs matching filter          │
└──────────────────────┴──────────────────────────────────────────┘
```

---

### Q42: What is the ESR Rule for compound indexes? ⭐🔥

```
ESR = Equality → Sort → Range

When creating a compound index, ORDER the fields as:
  1. EQUALITY fields first    (exact match: status = "active")
  2. SORT fields second       (ORDER BY: sort by date)
  3. RANGE fields last        (range: price > 100)

EXAMPLE:
  Query: db.products.find({ 
    category: "electronics",     // E: Equality
    price: { $gt: 100 }          // R: Range
  }).sort({ rating: -1 })        // S: Sort

  ✅ GOOD: createIndex({ category: 1, rating: -1, price: 1 })
  ❌ BAD:  createIndex({ price: 1, category: 1, rating: -1 })

WHY?
  Equality narrows to exact set → Sort uses index order → Range scans remainder
  If Range is before Sort → MongoDB can't use index for sorting → in-memory sort!
```

---

### Q43: What is a TTL Index?

```javascript
// TTL Index: Automatically delete documents after a time period

// Delete documents 24 hours after their "createdAt" field
db.sessions.createIndex(
    { createdAt: 1 },
    { expireAfterSeconds: 86400 }  // 86400 seconds = 24 hours
)

// Delete documents at a specific time (set expireAt field)
db.events.createIndex(
    { expireAt: 1 },
    { expireAfterSeconds: 0 }  // Delete when expireAt time is reached
)

// Document with custom expiration:
db.events.insertOne({
    event: "promo_code",
    expireAt: new Date("2024-12-31T23:59:59Z")  // Expires end of year
})

// NOTES:
// ⚠️ TTL thread runs every 60 seconds (not instant deletion)
// ⚠️ Only works on single-field indexes with Date type
// ⚠️ Cannot be compound index
// ✅ Perfect for: sessions, cache, temporary data, audit logs
```

---

### Q44: How do you identify and fix slow queries?

```javascript
// 1. ENABLE PROFILER (logs slow queries)
db.setProfilingLevel(1, { slowms: 100 })  // Log queries > 100ms

// 2. CHECK SLOW QUERIES
db.system.profile.find().sort({ ts: -1 }).limit(5)
// Look for: millis, docsExamined, keysExamined

// 3. EXPLAIN THE QUERY
db.orders.find({ status: "pending" }).explain("executionStats")
// Check: totalDocsExamined vs nReturned ratio
// Goal: ratio should be close to 1:1

// 4. CREATE APPROPRIATE INDEX
db.orders.createIndex({ status: 1, created_at: -1 })

// 5. CHECK INDEX USAGE
db.orders.aggregate([{ $indexStats: {} }])
// Find unused indexes → drop them!

// 6. USE $currentOp FOR RUNNING QUERIES
db.adminCommand({ currentOp: true, secs_running: { $gt: 5 } })
// Kill long-running query:
db.adminCommand({ killOp: 1, op: <opId> })
```

---

### Q45: What is the difference between Covered Query and Index-Only Scan?

```javascript
// COVERED QUERY: Query satisfied ENTIRELY by the index
// MongoDB never touches the actual documents!

// Create compound index
db.users.createIndex({ email: 1, name: 1 })

// ✅ COVERED: All queried/returned fields are in the index
db.users.find(
    { email: "alice@example.com" },
    { email: 1, name: 1, _id: 0 }     // ← Must exclude _id!
)
// explain() shows: "IXSCAN" (no "FETCH" stage) ⚡

// ❌ NOT COVERED: Requesting field not in index
db.users.find(
    { email: "alice@example.com" },
    { email: 1, name: 1, age: 1, _id: 0 }  // ← age not in index!
)
// explain() shows: "IXSCAN" → "FETCH" (must access document) 🐌
```

> ⚠️ **Must exclude `_id`** in projection for covered query (unless _id is in the index).

---

### Q46-Q50: Quick Performance Questions

**Q46: What is the `$hint` operator?**
```javascript
// Force MongoDB to use a specific index
db.orders.find({ status: "active", amount: { $gt: 100 } })
    .hint({ status: 1, amount: 1 })
// Use when optimizer picks wrong index. Avoid in production if possible.
```

**Q47: What is MongoDB's memory management?**
```
WiredTiger Cache: Stores frequently accessed data + indexes
  Default: max(256MB, 50% of RAM - 1GB)
  
  8GB RAM server: (8 - 1) * 0.5 = 3.5 GB cache
  32GB RAM server: (32 - 1) * 0.5 = 15.5 GB cache
  
  If working set > cache → constant disk reads → SLOW 💀
  Monitor: db.serverStatus().wiredTiger.cache
```

**Q48: What are MongoDB's read preferences?**
```javascript
// Where to route read queries in a replica set
// primary         → Always read from primary (default, strongest)
// primaryPreferred→ Primary if available, else secondary
// secondary       → Always from secondary (may be stale)
// secondaryPreferred → Secondary if available, else primary
// nearest         → Lowest network latency node

db.orders.find({}).readPref("secondaryPreferred")
```

**Q49: What is the `compact` command?**
```javascript
// Defragment a collection (reclaim disk space after many deletes)
db.runCommand({ compact: "orders" })
// ⚠️ Blocks the collection (use during maintenance window)
// Alternative: compact runs automatically on WiredTiger (mostly)
```

**Q50: How do you monitor MongoDB performance?**
```javascript
db.serverStatus()         // Overall server metrics
db.stats()                // Database statistics
db.collection.stats()     // Collection statistics
db.currentOp()            // Currently running operations

// Key metrics to watch:
// • opcounters (insert/query/update/delete rates)
// • connections.current (connection count)
// • mem.resident (memory usage)
// • wiredTiger.cache (cache hit ratio)
// • repl.lag (replication lag)
// • globalLock.activeClients (concurrent operations)

// Tools: mongostat, mongotop, Atlas UI, Prometheus + Grafana
```

---

# 🔄 SECTION 5: Replication & Sharding (Q51–Q60)

---

### Q51: What is a Replica Set? How does it work? ⭐

```
REPLICA SET: A group of MongoDB instances maintaining the SAME data.

  ┌──────────────┐
  │   PRIMARY    │ ← All writes go here
  │   (Leader)   │
  └──────┬───────┘
         │ Replication (oplog)
    ┌────┴────┐
    ↓         ↓
┌──────────┐ ┌──────────┐
│SECONDARY │ │SECONDARY │ ← Read replicas
│(Follower)│ │(Follower)│
└──────────┘ └──────────┘

COMPONENTS:
  • Primary: Receives ALL writes, replicates via oplog
  • Secondaries: Replicate from primary, can serve reads
  • Arbiter: Votes in elections, doesn't hold data (tiebreaker)
  
ELECTION:
  • Primary fails → secondaries vote → new primary elected
  • Requires MAJORITY (3 nodes → need 2 votes)
  • Election takes ~10-12 seconds (during this: no writes!)

MINIMUM: 3 members (or 2 data + 1 arbiter)
MAXIMUM: 50 members (7 voting max)
```

---

### Q52: What is the Oplog?

```
OPLOG (Operations Log): A capped collection that records ALL write operations.
Secondaries read the oplog to replicate data.

  Location: local.oplog.rs
  Type: Capped collection (circular buffer)

OPLOG ENTRY:
{
    "ts": Timestamp(1234567890, 1),    // Operation timestamp
    "op": "i",                          // Operation type
    "ns": "mydb.users",                 // Namespace
    "o": { name: "Alice", age: 30 }     // Document/operation
}

Operation types:
  "i" = insert
  "u" = update
  "d" = delete
  "c" = command (createCollection, etc.)
  "n" = no-op (heartbeat)

OPLOG SIZE:
  Default: 5% of free disk space (990MB minimum)
  Important: If oplog is too small and secondary falls too far behind,
  it must do a FULL RESYNC (very expensive!) 💀
  
  Set appropriately: 
  rs.conf() → members.oplogSizeMB
```

---

### Q53: What happens during a failover?

```
PRIMARY FAILURE TIMELINE:

  T=0s:   Primary stops responding
  T=2s:   Heartbeat failures detected by secondaries
  T=10s:  Election timeout reached
  T=10s:  Most up-to-date secondary calls for election
  T=12s:  Majority votes → new primary elected
  T=12s:  New primary starts accepting writes
  T=12-15s: Drivers detect new primary, reconnect

  DURING FAILOVER (10-12 seconds):
    ❌ NO writes accepted
    ✅ Reads from secondaries still work (if read preference allows)
    ⚠️ Client retries needed (use retryable writes!)

  AFTER FAILOVER:
    Old primary comes back → becomes SECONDARY
    If old primary had unreplicated writes → ROLLBACK
    Rolled-back writes saved to: rollback/<collection>/<timestamp>.bson
```

---

### Q54: What is Sharding in MongoDB? ⭐

```
SHARDING: Horizontal partitioning across multiple servers.

┌──────────────────────────────────────────────────────────────┐
│                     mongos (Router)                          │
│              Routes queries to correct shard                 │
└───────┬──────────────┬──────────────────┬───────────────────┘
        │              │                  │
   ┌────┴────┐   ┌────┴────┐       ┌────┴────┐
   │ Shard 1 │   │ Shard 2 │       │ Shard 3 │
   │ RS: {   │   │ RS: {   │       │ RS: {   │
   │  P, S, S│   │  P, S, S│       │  P, S, S│
   │ }       │   │ }       │       │ }       │
   │ Chunks: │   │ Chunks: │       │ Chunks: │
   │ A-F     │   │ G-O     │       │ P-Z     │
   └─────────┘   └─────────┘       └─────────┘
   
   Config Servers (Replica Set):
   Store metadata: chunk → shard mapping, shard key ranges

COMPONENTS:
  mongos:   Query router (stateless, can have multiple)
  Shard:    Each shard is a replica set
  Config:   Stores cluster metadata (must be replica set)
  Balancer: Moves chunks between shards for even distribution
```

---

### Q55: How do you choose a Shard Key? ⭐🔥

```
SHARD KEY: The field(s) that determine which shard a document lives on.
⚠️ CANNOT BE CHANGED after sharding (until MongoDB 5.0: resharding)

GOOD SHARD KEY PROPERTIES:
  ✅ High Cardinality: Many distinct values (user_id: millions)
  ✅ Even Distribution: No hot spots
  ✅ Query Isolation: Most queries target ONE shard
  ✅ Monotonically non-increasing: Avoid last-shard hot spot

SHARD KEY EXAMPLES:
  ✅ { user_id: 1 }              → Good: high cardinality, query isolation
  ✅ { user_id: "hashed" }       → Good: even distribution
  ✅ { tenant_id: 1, _id: 1 }   → Good: multi-tenant, compound
  ❌ { created_at: 1 }           → Bad: monotonically increasing → hot spot!
  ❌ { status: 1 }               → Bad: low cardinality (only 3-4 values)
  ❌ { country: 1 }              → Bad: uneven (50% might be one country)

STRATEGIES:
  Hashed:    Even distribution, no range queries
  Ranged:    Range queries work, risk of hot spots
  Compound:  Best of both (zone-based sharding)
```

---

### Q56: What is the Balancer?

```
BALANCER: Background process that moves CHUNKS between shards 
to maintain even data distribution.

TRIGGER: Imbalance exceeds threshold
  Small: > 2 chunk difference between max and min shard
  Large: > 8 chunk difference

PROCESS:
  1. Balancer detects imbalance
  2. Selects chunks to move (from most-loaded to least-loaded)
  3. Moves chunk data to destination shard
  4. Updates config server metadata
  5. Source shard deletes migrated data

CONTROL:
  sh.stopBalancer()                  // Stop balancer
  sh.startBalancer()                 // Start balancer
  sh.setBalancerState(false)         // Disable
  
  // Schedule balancer window (e.g., during off-peak hours)
  db.settings.update(
      { _id: "balancer" },
      { $set: { activeWindow: { start: "02:00", stop: "06:00" } } }
  )
```

---

### Q57: What is a Chunk in MongoDB sharding?

```
CHUNK: A contiguous range of shard key values.

  Shard Key: user_id (range-based)
  
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Chunk 1  │ │ Chunk 2  │ │ Chunk 3  │ │ Chunk 4  │
  │ 1-1000   │ │1001-2000 │ │2001-3000 │ │3001-4000 │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘
    Shard A      Shard A      Shard B      Shard B

DEFAULT CHUNK SIZE: 128 MB (was 64 MB before 6.0)

CHUNK SPLITTING:
  When a chunk exceeds max size → auto-split into 2 chunks
  Split happens at the MEDIAN shard key value
  
  Chunk [1-1000] grows > 128MB → Split:
    Chunk A: [1-500]
    Chunk B: [501-1000]

JUMBO CHUNKS: Chunks that can't be split (all docs have same shard key value)
  → Cannot be balanced! → Hot spot! 💀
  → This is why high cardinality shard keys are important
```

---

### Q58: What is a Targeted Query vs Scatter-Gather?

```
TARGETED QUERY (efficient):
  Query includes the SHARD KEY → mongos sends to ONE shard
  
  Shard key: { user_id: 1 }
  Query: db.orders.find({ user_id: 42 })
  → mongos knows user_id 42 is on Shard 2 → sends ONLY to Shard 2 ⚡

SCATTER-GATHER (expensive):
  Query does NOT include shard key → mongos sends to ALL shards
  
  Shard key: { user_id: 1 }
  Query: db.orders.find({ product: "Widget" })
  → mongos doesn't know which shard → broadcasts to ALL shards 🐌
  → Each shard executes → mongos merges results

BROADCAST OPERATIONS:
  db.collection.find({})              → All shards
  db.collection.find({ non_sk: val }) → All shards
  db.collection.count()               → All shards
  
  ALWAYS try to include the shard key in queries!
```

---

### Q59-Q60: Quick Replication & Sharding Questions

**Q59: What is Zone-Based Sharding?**
```javascript
// Assign shard key ranges to specific shards (geographic routing)
sh.addShardTag("shard0001", "US")
sh.addShardTag("shard0002", "EU")

sh.addTagRange("mydb.users", 
    { country: "US" }, { country: "US\uffff" }, "US")
sh.addTagRange("mydb.users",
    { country: "DE" }, { country: "FR\uffff" }, "EU")
// US users → US shard, European users → EU shard
// Compliance: GDPR data stays in EU! ✅
```

**Q60: What is the difference between rs.initiate() and rs.add()?**
```javascript
// rs.initiate(): Initialize a NEW replica set
rs.initiate({
    _id: "myReplicaSet",
    members: [
        { _id: 0, host: "mongo1:27017" },
        { _id: 1, host: "mongo2:27017" },
        { _id: 2, host: "mongo3:27017" }
    ]
})

// rs.add(): Add a member to EXISTING replica set
rs.add("mongo4:27017")
rs.add({ host: "mongo5:27017", priority: 0, hidden: true })  // Hidden member

// rs.remove(): Remove a member
rs.remove("mongo4:27017")
```

---

# 🔐 SECTION 6: Transactions, Security & Operations (Q61–Q70)

---

### Q61: Does MongoDB support ACID Transactions?

```javascript
// YES! Since MongoDB 4.0 (replica sets) and 4.2 (sharded clusters)

const session = client.startSession();
session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" },
    readPreference: "primary"
});

try {
    const orders = session.getDatabase("shop").orders;
    const inventory = session.getDatabase("shop").inventory;
    
    // Multiple operations — all or nothing!
    orders.insertOne({ item: "widget", qty: 1, price: 25 }, { session });
    inventory.updateOne(
        { item: "widget" },
        { $inc: { stock: -1 } },
        { session }
    );
    
    session.commitTransaction();   // ✅ Both committed
} catch (error) {
    session.abortTransaction();    // ❌ Both rolled back
} finally {
    session.endSession();
}

// LIMITATIONS:
// ⚠️ 60-second timeout (default, configurable)
// ⚠️ 16MB oplog entry limit per transaction
// ⚠️ Performance overhead (~10-20% slower than non-transactional)
// ⚠️ Should be SHORT — not for long-running batch operations
// ⚠️ Still: design schema to avoid needing transactions when possible!
```

---

### Q62: What is MongoDB's security model?

```
┌─────────────────────────────────────────────────────────────┐
│                MONGODB SECURITY LAYERS                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. AUTHENTICATION (Who are you?)                          │
│     • SCRAM-SHA-256 (default)                              │
│     • x.509 Certificate                                    │
│     • LDAP (Enterprise)                                    │
│     • Kerberos (Enterprise)                                │
│                                                             │
│  2. AUTHORIZATION (What can you do?)                       │
│     • Role-Based Access Control (RBAC)                     │
│     • Built-in roles: read, readWrite, dbAdmin, root       │
│     • Custom roles                                         │
│                                                             │
│  3. ENCRYPTION                                             │
│     • In-transit: TLS/SSL                                  │
│     • At-rest: WiredTiger encryption (Enterprise)          │
│     • Field-Level: Client-Side Field Level Encryption      │
│       (CSFLE) — encrypt specific fields! ⭐                │
│                                                             │
│  4. NETWORK                                                │
│     • Bind to specific IP (not 0.0.0.0!)                  │
│     • Firewall rules / VPC peering                        │
│     • IP whitelist (Atlas)                                 │
│                                                             │
│  5. AUDITING (Enterprise)                                  │
│     • Log authentication, authorization, CRUD events       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Q63: What is Client-Side Field Level Encryption (CSFLE)?

```
CSFLE: Encrypt specific fields BEFORE they reach the server.
Even DBAs and MongoDB Atlas can't read the encrypted data!

Application                 MongoDB Server
┌──────────┐               ┌──────────────┐
│ SSN: 123 │──encrypt──→   │ SSN: Binary  │
│          │               │ (encrypted)  │
│ Name: Ali│──plaintext─→  │ Name: Alice  │
└──────────┘               └──────────────┘

// Encrypted document on server:
{
    name: "Alice",                          // Readable
    email: "alice@example.com",             // Readable  
    ssn: Binary("encrypted_blob..."),       // 🔒 Unreadable on server!
    credit_card: Binary("encrypted_blob...") // 🔒 Unreadable on server!
}

MODES:
  Deterministic: Same input → same ciphertext (equality queries work)
  Random:        Same input → different ciphertext (no queries, more secure)

USE CASES:
  ✅ PII (SSN, credit cards, health records)
  ✅ Regulatory compliance (GDPR, HIPAA, PCI-DSS)
  ✅ Multi-tenant data isolation
```

---

### Q64: What is Change Streams?

```javascript
// CHANGE STREAMS: Real-time notifications for data changes
// Like database triggers, but at the application level!

const changeStream = db.collection("orders").watch([
    { $match: { "fullDocument.status": "shipped" } }
]);

changeStream.on("change", (change) => {
    console.log("Operation:", change.operationType);  // insert/update/delete
    console.log("Document:", change.fullDocument);
    console.log("Update:", change.updateDescription);
    
    // React to change: send notification, update cache, etc.
    sendNotification(change.fullDocument.customer_email);
});

// Watch entire database
const dbStream = db.watch();

// Watch entire deployment (all databases)
const clusterStream = client.watch();

// Resume after disconnect (fault-tolerant!)
const stream = db.collection("orders").watch([], {
    resumeAfter: lastSeenResumeToken  // Continue from where you left off
});

// USE CASES:
// ✅ Real-time notifications (order status updates)
// ✅ Cache invalidation (update Redis when MongoDB changes)
// ✅ Event-driven architecture (trigger microservices)
// ✅ Data synchronization (replicate to search index)
// ✅ Audit logging
```

---

### Q65: How do you back up MongoDB?

```
BACKUP METHODS:

┌──────────────────────────────────────────────────────────────┐
│ 1. mongodump / mongorestore (Logical Backup)               │
│    mongodump --uri="mongodb://..." --out=/backup/           │
│    mongorestore --uri="mongodb://..." /backup/              │
│    ✅ Simple, portable                                       │
│    ❌ Slow on large databases (reads all data)              │
│    ❌ Not point-in-time consistent (unless --oplog)         │
├──────────────────────────────────────────────────────────────┤
│ 2. File System Snapshot (Physical Backup)                   │
│    • LVM snapshot, EBS snapshot, ZFS snapshot               │
│    ✅ Fast (instant snapshot)                                │
│    ✅ Consistent with journaling enabled                     │
│    ❌ Must lock or use journaling                           │
├──────────────────────────────────────────────────────────────┤
│ 3. MongoDB Atlas Automated Backups                          │
│    ✅ Continuous backup with point-in-time recovery          │
│    ✅ Fully managed, no maintenance                         │
│    ✅ Restore to any second in last N days                   │
│    ❌ Only on Atlas (cloud)                                 │
├──────────────────────────────────────────────────────────────┤
│ 4. Ops Manager / Cloud Manager (Enterprise)                 │
│    ✅ Scheduled, point-in-time, queryable backups            │
│    ✅ On-premise enterprise solution                        │
└──────────────────────────────────────────────────────────────┘
```

---

### Q66-Q70: Quick Operations Questions

**Q66: What is the `$merge` stage?**
```javascript
// Write aggregation results to a collection (upsert/replace/merge)
db.orders.aggregate([
    { $group: { _id: "$customer_id", total: { $sum: "$amount" } } },
    { $merge: {
        into: "customer_totals",
        on: "_id",
        whenMatched: "replace",
        whenNotMatched: "insert"
    }}
])
// Perfect for: materialized views, pre-computed reports
```

**Q67: What is `db.currentOp()` used for?**
```javascript
// Show currently running operations (like SQL's SHOW PROCESSLIST)
db.currentOp({ "secs_running": { $gt: 5 } })

// Kill a long-running operation
db.adminCommand({ killOp: 1, op: 12345 })
```

**Q68: What are Retryable Writes?**
```javascript
// MongoDB 3.6+: Automatically retry certain write operations once
// on transient failures (network error, primary election)
const client = new MongoClient(uri, { retryWrites: true });
// Retryable: insertOne, updateOne, deleteOne, findOneAndUpdate, bulkWrite
// NOT retryable: updateMany, deleteMany, aggregate with $out
```

**Q69: What is the `$jsonSchema` validator?**
```javascript
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "email"],
            properties: {
                name: { bsonType: "string", description: "must be a string" },
                email: { bsonType: "string", pattern: "^.+@.+$" },
                age: { bsonType: "int", minimum: 0, maximum: 150 }
            }
        }
    },
    validationLevel: "strict",    // or "moderate" (skip existing invalid docs)
    validationAction: "error"     // or "warn" (log but allow)
})
```

**Q70: How do you upgrade MongoDB version with zero downtime?**
```
ROLLING UPGRADE (Zero Downtime):
  1. Upgrade secondaries one at a time
  2. Step down primary (rs.stepDown())
  3. Upgrade old primary (now secondary)
  4. Set feature compatibility version
  
  db.adminCommand({ setFeatureCompatibilityVersion: "7.0" })
  
  ⚠️ Always upgrade one major version at a time (5.0 → 6.0 → 7.0)
  ⚠️ Test in staging first!
  ⚠️ Read release notes for breaking changes
```

---

## 🔑 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONGODB INTERVIEW SURVIVAL KIT                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Schema design is the #1 interview topic — know embed vs ref   │
│  2. Aggregation pipeline — practice $group, $lookup, $unwind      │
│  3. ESR Rule for indexes — Equality → Sort → Range                │
│  4. Replica sets: 3 minimum, oplog, automatic failover            │
│  5. Sharding: shard key selection is CRITICAL                     │
│  6. Transactions exist (4.0+) but design to avoid needing them    │
│  7. explain() is your best friend — learn to read it              │
│  8. Change Streams for event-driven architectures                 │
│  9. Write/Read concerns control consistency guarantees            │
│  10. MongoDB is NOT schemaless — it's flexible schema             │
│                                                                     │
│  MOST COMMON MISTAKE: Treating MongoDB like a relational DB       │
│  → Embedding is the default, referencing is the exception!        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Link |
|-------------|------|
| SQL Practice Problems | [Chapter 8.4 — 50 Must-Solve Problems](./04-SQL-Practice-Problems.md) |
| Quick Cheat Sheets | [Chapter 8.5 — Cheat Sheets](./05-Cheat-Sheets.md) |

---

> **"MongoDB isn't about forgetting relational databases. It's about rethinking how you model data when relationships aren't the center of your universe."** 🍃
