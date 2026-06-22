# 🍃 Chapter 3B.3 — MongoDB CRUD — The Complete Guide

> **Level:** 🟢 Beginner | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4–5 hours
> **Prerequisites:** Chapter 3B.2 (MongoDB Installation & Setup)

> **"CRUD isn't glamorous, but it's the bread and butter of every application ever built. Master it, and you master 80% of daily MongoDB work."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- **Insert** single and multiple documents like a pro
- **Query** with every operator MongoDB offers (comparison, logical, element, array, regex)
- **Update** documents with surgical precision ($set, $inc, $push, $pull, and more)
- **Delete** documents safely
- Use **Projections** to return only the fields you need
- Master **Cursors**, sorting, limiting, and pagination
- Know the **Bulk Operations** pattern for high-performance writes

---

## 🧪 Setup — Our Practice Database

Throughout this chapter, we'll work with an **e-commerce** database. Run this in `mongosh` to set up:

```javascript
use ecommerce

// We'll populate data as we learn each operation.
// By the end, you'll have a fully loaded practice database.
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PART 1: CREATE (Insert Operations)
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### 1.1 insertOne() — Insert a Single Document

```javascript
// Syntax: db.collection.insertOne(document, options)

// ── Basic Insert ──
db.users.insertOne({
  name: "Ritesh Singh",
  email: "ritesh@example.com",
  age: 28,
  city: "Delhi",
  isActive: true,
  joinedAt: new Date()
})

// Output:
// {
//   acknowledged: true,
//   insertedId: ObjectId('507f1f77bcf86cd799439011')
// }
```

```javascript
// ── Insert with Custom _id ──
db.users.insertOne({
  _id: "user_001",              // ← You can set your own _id!
  name: "Priya Sharma",
  email: "priya@example.com",
  age: 25
})

// ⚠️ If _id already exists → DuplicateKeyError
db.users.insertOne({ _id: "user_001", name: "Another Person" })
// ERROR: E11000 duplicate key error collection: ecommerce.users
```

```javascript
// ── Insert a RICH Document (nested objects + arrays) ──
db.products.insertOne({
  name: "iPhone 15 Pro",
  brand: "Apple",
  price: 134900,
  currency: "INR",
  category: "Electronics",
  specs: {                        // ← Embedded document
    display: "6.1 inch OLED",
    chip: "A17 Pro",
    storage: ["128GB", "256GB", "512GB", "1TB"],
    camera: {
      main: "48MP",
      ultrawide: "12MP",
      telephoto: "12MP"
    }
  },
  colors: ["Natural Titanium", "Blue Titanium", "White Titanium", "Black Titanium"],
  tags: ["smartphone", "premium", "5G", "iOS"],
  stock: 250,
  ratings: {
    average: 4.7,
    count: 12589
  },
  createdAt: new Date(),
  updatedAt: new Date()
})
```

> 💡 **Pro Tip**: MongoDB creates the collection automatically on first insert. No `CREATE TABLE` needed!

---

### 1.2 insertMany() — Insert Multiple Documents

```javascript
// Syntax: db.collection.insertMany([doc1, doc2, ...], options)

db.users.insertMany([
  {
    name: "Amit Verma",
    email: "amit@example.com",
    age: 32,
    city: "Mumbai",
    skills: ["Java", "Spring Boot"],
    isActive: true
  },
  {
    name: "Sneha Patel",
    email: "sneha@example.com",
    age: 27,
    city: "Bangalore",
    skills: ["Python", "Django", "PostgreSQL"],
    isActive: true
  },
  {
    name: "Ravi Kumar",
    email: "ravi@example.com",
    age: 35,
    city: "Chennai",
    skills: ["JavaScript", "React", "MongoDB"],
    isActive: false
  },
  {
    name: "Neha Gupta",
    email: "neha@example.com",
    age: 29,
    city: "Delhi",
    skills: ["C#", ".NET", "SQL Server"],
    isActive: true
  },
  {
    name: "Karan Mehta",
    email: "karan@example.com",
    age: 24,
    city: "Pune",
    skills: ["Go", "Docker", "Kubernetes"],
    isActive: true
  }
])

// Output:
// {
//   acknowledged: true,
//   insertedIds: {
//     '0': ObjectId('...'),
//     '1': ObjectId('...'),
//     ...
//   }
// }
```

### 1.3 Ordered vs Unordered Inserts

```javascript
// ── ORDERED (default: true) ──
// Stops at first error. Documents after the error are NOT inserted.
db.users.insertMany([
  { _id: 1, name: "A" },
  { _id: 1, name: "B" },    // ❌ Duplicate _id → STOPS HERE
  { _id: 2, name: "C" }     // ❌ Never inserted
], { ordered: true })

// ── UNORDERED (ordered: false) ──
// Continues past errors. Inserts everything that doesn't fail.
db.users.insertMany([
  { _id: 1, name: "A" },    // ✅ Inserted (or fails if exists)
  { _id: 1, name: "B" },    // ❌ Fails (duplicate)
  { _id: 2, name: "C" }     // ✅ Still inserted!
], { ordered: false })
```

```
When to use what?

  ordered: true (default)
  → Use when INSERT ORDER MATTERS
  → Use when later docs DEPEND on earlier ones
  → Slower (can't parallelize)

  ordered: false
  → Use for BULK LOADING data
  → Use when you want to insert as many as possible
  → Faster (MongoDB can parallelize)
  → Best for initial data migration
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PART 2: READ (Query Operations)
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

> This is the BIGGEST section. MongoDB's query language is incredibly powerful.

### 2.1 find() and findOne() — The Basics

```javascript
// ── findOne() — Returns FIRST matching document ──
db.users.findOne()                              // First doc in collection
db.users.findOne({ name: "Ritesh Singh" })      // First matching doc
db.users.findOne({ age: 28, city: "Delhi" })    // Multiple conditions (AND)

// ── find() — Returns a CURSOR (lazy iterator of results) ──
db.users.find()                                 // All documents
db.users.find({ city: "Delhi" })                // All users in Delhi
db.users.find({ isActive: true })               // All active users

// Exact match on nested field (use dot notation)
db.products.find({ "specs.chip": "A17 Pro" })

// Exact match on array element
db.users.find({ skills: "Python" })             // Any user with "Python" in skills array
```

---

### 2.2 Comparison Operators — The Power Tools

```javascript
// ┌──────────────────────────────────────────────────────────────┐
// │  COMPARISON OPERATORS                                        │
// ├───────────┬──────────────────────────────────────────────────┤
// │ Operator  │ Meaning                                          │
// ├───────────┼──────────────────────────────────────────────────┤
// │ $eq       │ Equal to (implicit when using { field: value })  │
// │ $ne       │ Not equal to                                     │
// │ $gt       │ Greater than                                     │
// │ $gte      │ Greater than or equal to                         │
// │ $lt       │ Less than                                        │
// │ $lte      │ Less than or equal to                            │
// │ $in       │ Matches ANY value in array                       │
// │ $nin      │ Matches NONE of the values in array              │
// └───────────┴──────────────────────────────────────────────────┘

// ── $eq (explicit vs implicit) ──
db.users.find({ age: 28 })                       // Implicit $eq
db.users.find({ age: { $eq: 28 } })              // Explicit $eq (same result)

// ── $ne (not equal) ──
db.users.find({ city: { $ne: "Delhi" } })         // Everyone NOT in Delhi

// ── $gt, $gte (greater than) ──
db.users.find({ age: { $gt: 25 } })               // Age > 25
db.users.find({ age: { $gte: 28 } })              // Age >= 28

// ── $lt, $lte (less than) ──
db.users.find({ age: { $lt: 30 } })               // Age < 30
db.users.find({ age: { $lte: 27 } })              // Age <= 27

// ── RANGE (combine $gt and $lt) ──
db.users.find({ age: { $gt: 25, $lt: 35 } })      // 25 < age < 35
db.users.find({ age: { $gte: 25, $lte: 35 } })    // 25 ≤ age ≤ 35

// ── $in (match any value in list) ──
db.users.find({ city: { $in: ["Delhi", "Mumbai", "Bangalore"] } })
// = WHERE city IN ('Delhi', 'Mumbai', 'Bangalore')

// ── $nin (match none in list) ──
db.users.find({ city: { $nin: ["Delhi", "Chennai"] } })
// = WHERE city NOT IN ('Delhi', 'Chennai')
```

---

### 2.3 Logical Operators — Combine Conditions

```javascript
// ┌──────────────────────────────────────────────────────────────┐
// │  LOGICAL OPERATORS                                           │
// ├───────────┬──────────────────────────────────────────────────┤
// │ $and      │ All conditions must be true                      │
// │ $or       │ At least one condition must be true              │
// │ $not      │ Inverts the condition                            │
// │ $nor      │ NONE of the conditions should be true            │
// └───────────┴──────────────────────────────────────────────────┘

// ── $and (explicit) ──
db.users.find({
  $and: [
    { age: { $gte: 25 } },
    { city: "Delhi" },
    { isActive: true }
  ]
})

// ── $and (implicit — just put conditions in same object) ──
db.users.find({
  age: { $gte: 25 },
  city: "Delhi",
  isActive: true
})
// ↑ Same as explicit $and! MongoDB uses implicit AND by default.

// ⚠️ When do you NEED explicit $and?
// When you have MULTIPLE conditions on the SAME FIELD:
db.users.find({
  $and: [
    { age: { $gt: 20 } },
    { age: { $lt: 30 } }
  ]
})
// Note: { age: { $gt: 20, $lt: 30 } } also works for this case

// ── $or ──
db.users.find({
  $or: [
    { city: "Delhi" },
    { city: "Mumbai" }
  ]
})
// = WHERE city = 'Delhi' OR city = 'Mumbai'
// (for this case, $in is more efficient)

// More complex $or:
db.users.find({
  $or: [
    { age: { $lt: 25 } },
    { skills: "Python" }
  ]
})
// Users who are under 25 OR know Python

// ── Combining $and and $or ──
db.users.find({
  isActive: true,                   // AND
  $or: [                            // (city = Delhi OR age > 30)
    { city: "Delhi" },
    { age: { $gt: 30 } }
  ]
})
// = WHERE isActive = true AND (city = 'Delhi' OR age > 30)

// ── $not (invert a condition) ──
db.users.find({ age: { $not: { $gt: 30 } } })
// Users whose age is NOT greater than 30 (includes docs where age doesn't exist!)

// ── $nor (none of these should be true) ──
db.users.find({
  $nor: [
    { city: "Delhi" },
    { isActive: false }
  ]
})
// Users who are NOT in Delhi AND NOT inactive
```

---

### 2.4 Element Operators — Check Field Existence & Type

```javascript
// ┌───────────┬──────────────────────────────────────────────────┐
// │ $exists   │ Does the field exist?                            │
// │ $type     │ Is the field a specific BSON type?               │
// └───────────┴──────────────────────────────────────────────────┘

// ── $exists ──
db.users.find({ phone: { $exists: true } })      // Has a "phone" field
db.users.find({ phone: { $exists: false } })     // Does NOT have "phone" field

// ── $type ──
db.users.find({ age: { $type: "int" } })          // age is an integer
db.users.find({ age: { $type: "string" } })       // age is a string (data issue!)
db.users.find({ name: { $type: "string" } })      // name is a string

// Multiple types:
db.users.find({ age: { $type: ["int", "double"] } })  // age is int OR double

// Useful BSON type names:
// "double", "string", "object", "array", "binData", "objectId",
// "bool", "date", "null", "regex", "int", "long", "decimal"
```

> 💡 **Pro Tip**: `$exists` + `$type` is your **data quality auditing toolkit**. Use it to find documents with missing fields or wrong types — especially in schema-flexible databases where bad data sneaks in.

---

### 2.5 Array Operators — Query Arrays Like a Pro

```javascript
// Let's first insert some products with array fields:
db.products.insertMany([
  {
    name: "MacBook Pro 16",
    tags: ["laptop", "apple", "premium", "M3"],
    colors: ["Silver", "Space Black"],
    reviews: [
      { user: "Alice", rating: 5, comment: "Amazing!" },
      { user: "Bob", rating: 4, comment: "Great but expensive" },
      { user: "Charlie", rating: 5, comment: "Best laptop ever" }
    ]
  },
  {
    name: "ThinkPad X1",
    tags: ["laptop", "lenovo", "business"],
    colors: ["Black"],
    reviews: [
      { user: "Dave", rating: 4, comment: "Solid build" },
      { user: "Eve", rating: 3, comment: "Keyboard is great" }
    ]
  }
])
```

```javascript
// ┌───────────────┬──────────────────────────────────────────────┐
// │ ARRAY OPERATORS                                              │
// ├───────────────┬──────────────────────────────────────────────┤
// │ (no operator) │ Match element in array (implicit)            │
// │ $all          │ Array must contain ALL specified elements     │
// │ $elemMatch    │ At least one element matches ALL conditions  │
// │ $size         │ Array has exactly N elements                  │
// └───────────────┴──────────────────────────────────────────────┘

// ── Implicit Array Match ──
db.products.find({ tags: "premium" })
// Finds docs where "tags" array CONTAINS "premium"

// ── $all — Must contain ALL specified values ──
db.products.find({ tags: { $all: ["laptop", "premium"] } })
// Must have BOTH "laptop" AND "premium" in tags
// Order doesn't matter

// ── $size — Array length must be exactly N ──
db.products.find({ colors: { $size: 1 } })
// Products with exactly 1 color option

// ⚠️ $size doesn't support ranges! For that, use aggregation.

// ── $elemMatch — Complex conditions on array elements ──
// Find products with a review where rating >= 5 AND user is "Alice"
db.products.find({
  reviews: {
    $elemMatch: {
      rating: { $gte: 5 },
      user: "Alice"
    }
  }
})
```

```
Why $elemMatch matters — A CRITICAL distinction:

// ❌ WITHOUT $elemMatch (WRONG for multi-condition):
db.products.find({
  "reviews.rating": { $gte: 5 },
  "reviews.user": "Bob"
})
// This finds docs where:
//   ANY review has rating >= 5   (Alice's review: rating 5 ✅)
//   AND ANY review has user "Bob" (Bob's review ✅)
// → MATCHES! But Bob's rating is 4, not 5!
// → The conditions matched DIFFERENT array elements! 🐛

// ✅ WITH $elemMatch (CORRECT):
db.products.find({
  reviews: {
    $elemMatch: {
      rating: { $gte: 5 },
      user: "Bob"
    }
  }
})
// This finds docs where THE SAME review element has:
//   rating >= 5 AND user = "Bob"
// → NO MATCH (correct! Bob's rating is 4)
```

> 💡 **Pro Tip**: Whenever you have **multiple conditions on elements of an array of objects**, use `$elemMatch`. Without it, conditions can match across different elements, giving wrong results.

---

### 2.6 Evaluation Operators — Regex, Expr, and More

```javascript
// ┌───────────┬──────────────────────────────────────────────────┐
// │ $regex    │ Regular expression pattern matching              │
// │ $expr     │ Use aggregation expressions in queries           │
// │ $text     │ Full-text search (requires text index)           │
// │ $mod      │ Modulo operation                                 │
// │ $where    │ JavaScript expression (⚠️ avoid — slow!)        │
// └───────────┴──────────────────────────────────────────────────┘

// ── $regex — Pattern Matching ──
// Names starting with "R"
db.users.find({ name: { $regex: /^R/ } })

// Names containing "kumar" (case-insensitive)
db.users.find({ name: { $regex: /kumar/i } })

// Email from gmail
db.users.find({ email: { $regex: /@gmail\.com$/ } })

// Alternative syntax:
db.users.find({ name: { $regex: "^R", $options: "i" } })

// Common regex patterns:
// /^text/     → Starts with "text"
// /text$/     → Ends with "text"
// /text/      → Contains "text"
// /text/i     → Case-insensitive
// /^text$/    → Exactly "text"

// ⚠️ PERFORMANCE WARNING: 
// Regex that starts with .* or doesn't anchor to ^ 
// CANNOT use indexes efficiently → FULL COLLECTION SCAN!
// /^Ritesh/   ← ✅ Can use index (prefix match)
// /Ritesh/    ← ❌ Cannot use index (scan everything)

// ── $expr — Compare fields within the same document ──
// Products where stock < reorderLevel (field vs field comparison)
db.products.insertOne({
  name: "Widget",
  stock: 5,
  reorderLevel: 10
})

db.products.find({
  $expr: { $lt: ["$stock", "$reorderLevel"] }
})
// "Find products where stock is less than reorderLevel"
// This compares TWO FIELDS in the SAME document

// ── $mod — Modulo ──
db.users.find({ age: { $mod: [2, 0] } })    // Age is even
db.users.find({ age: { $mod: [5, 0] } })    // Age is divisible by 5
```

---

### 2.7 Projections — Return Only What You Need

```javascript
// Projection = Selecting which fields to return
// Like SQL's SELECT name, email FROM users (instead of SELECT *)

// Syntax: db.collection.find(query, projection)

// ── Include specific fields (1 = include) ──
db.users.find(
  { city: "Delhi" },
  { name: 1, email: 1 }
)
// Returns: { _id: ..., name: "Ritesh", email: "ritesh@..." }
// Note: _id is ALWAYS included by default

// ── Exclude _id ──
db.users.find(
  { city: "Delhi" },
  { name: 1, email: 1, _id: 0 }
)
// Returns: { name: "Ritesh", email: "ritesh@..." }

// ── Exclude specific fields (0 = exclude) ──
db.users.find(
  {},
  { password: 0, ssn: 0 }
)
// Returns everything EXCEPT password and ssn

// ⚠️ RULE: You can't MIX include and exclude (except _id)!
db.users.find({}, { name: 1, age: 0 })    // ❌ ERROR!
db.users.find({}, { name: 1, _id: 0 })    // ✅ OK (_id is special)

// ── Projection on Nested Fields ──
db.products.find(
  { name: "iPhone 15 Pro" },
  { "specs.chip": 1, "specs.storage": 1, price: 1 }
)

// ── $slice — Limit array elements ──
db.products.find(
  { name: "MacBook Pro 16" },
  { reviews: { $slice: 2 } }            // First 2 reviews only
)

db.products.find(
  { name: "MacBook Pro 16" },
  { reviews: { $slice: -1 } }           // Last review only
)

db.products.find(
  { name: "MacBook Pro 16" },
  { reviews: { $slice: [1, 2] } }       // Skip 1, take 2 (pagination!)
)

// ── $elemMatch in Projection ──
db.products.find(
  { name: "MacBook Pro 16" },
  { reviews: { $elemMatch: { rating: { $gte: 5 } } } }
)
// Returns only the FIRST review with rating >= 5
```

---

### 2.8 Cursors — Sorting, Limiting, Skipping, Counting

```javascript
// find() returns a CURSOR — a pointer to the result set.
// Chain methods on the cursor:

// ── sort() — Order results ──
db.users.find().sort({ age: 1 })          // Ascending (youngest first)
db.users.find().sort({ age: -1 })         // Descending (oldest first)
db.users.find().sort({ city: 1, age: -1 })  // City A→Z, then age high→low

// ── limit() — Cap the number of results ──
db.users.find().limit(5)                  // First 5 documents
db.users.find().sort({ age: -1 }).limit(1)  // Oldest user

// ── skip() — Skip N documents ──
db.users.find().skip(10)                  // Skip first 10

// ── PAGINATION PATTERN ──
const pageSize = 10;
const pageNumber = 3;   // Page 3

db.users.find()
  .sort({ _id: 1 })
  .skip((pageNumber - 1) * pageSize)     // Skip 20
  .limit(pageSize)                        // Take 10

// Page 1: skip(0).limit(10)   → docs 1-10
// Page 2: skip(10).limit(10)  → docs 11-20
// Page 3: skip(20).limit(10)  → docs 21-30

// ⚠️ WARNING: skip() is SLOW for large offsets!
// For large datasets, use RANGE-BASED pagination instead:
// Get last _id from previous page, then:
db.users.find({ _id: { $gt: lastSeenId } })
  .sort({ _id: 1 })
  .limit(pageSize)

// ── countDocuments() — Count matching documents ──
db.users.countDocuments()                             // Total users
db.users.countDocuments({ city: "Delhi" })             // Users in Delhi
db.users.countDocuments({ age: { $gte: 25, $lte: 35 } })  // Age 25-35

// ── estimatedDocumentCount() — Faster but approximate ──
db.users.estimatedDocumentCount()
// Uses metadata (O(1)) instead of scanning — much faster for huge collections

// ── distinct() — Unique values for a field ──
db.users.distinct("city")
// Output: ["Delhi", "Mumbai", "Bangalore", "Chennai", "Pune"]

db.users.distinct("city", { isActive: true })
// Unique cities of active users only
```

```
Execution Order (important!):
  
  db.users.find(query)    ← 1. Filter documents
    .sort({ age: 1 })     ← 2. Sort the filtered results
    .skip(20)              ← 3. Skip first 20
    .limit(10)             ← 4. Take next 10

  Note: MongoDB optimizes this internally.
  Even if you write .limit().sort(), MongoDB ALWAYS sorts first.
  
  Execution: filter → sort → skip → limit
  (Regardless of the order you chain them!)
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PART 3: UPDATE Operations
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### 3.1 Update Methods Overview

```
┌───────────────────────┬──────────────────────────────────────┐
│  Method               │  What It Does                        │
├───────────────────────┼──────────────────────────────────────┤
│  updateOne()          │  Update FIRST matching document       │
│  updateMany()         │  Update ALL matching documents        │
│  replaceOne()         │  Replace entire document              │
│  findOneAndUpdate()   │  Update & return the document         │
│  findOneAndReplace()  │  Replace & return the document        │
└───────────────────────┴──────────────────────────────────────┘
```

### 3.2 Field Update Operators — The Complete Toolkit

```javascript
// ┌──────────────────────────────────────────────────────────────┐
// │  FIELD UPDATE OPERATORS                                      │
// ├───────────┬──────────────────────────────────────────────────┤
// │ $set      │ Set field value (create if doesn't exist)        │
// │ $unset    │ Remove a field                                   │
// │ $inc      │ Increment by value (or decrement with negative)  │
// │ $mul      │ Multiply by value                                │
// │ $min      │ Update only if new value is less than current    │
// │ $max      │ Update only if new value is greater than current │
// │ $rename   │ Rename a field                                   │
// │ $setOnInsert │ Set only during upsert insert (not update)   │
// │ $currentDate │ Set field to current date                    │
// └───────────┴──────────────────────────────────────────────────┘

// ── $set — Set/Update field values ──
db.users.updateOne(
  { name: "Ritesh Singh" },           // Filter: who to update
  { $set: { age: 29, city: "Noida" } }  // Update: what to change
)
// Changes age to 29 and city to "Noida"
// If "city" didn't exist, it gets CREATED

// ── $set nested fields ──
db.products.updateOne(
  { name: "iPhone 15 Pro" },
  { $set: { "specs.chip": "A18 Pro", "specs.camera.main": "50MP" } }
)

// ── $unset — Remove a field ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $unset: { tempField: "" } }        // Value doesn't matter, "" is convention
)

// ── $inc — Increment / Decrement ──
db.products.updateOne(
  { name: "iPhone 15 Pro" },
  { $inc: { stock: -1 } }             // Decrease stock by 1 (sold one!)
)

db.users.updateOne(
  { name: "Ritesh Singh" },
  { $inc: { age: 1 } }                // Happy birthday! 🎂
)

// ── $mul — Multiply ──
db.products.updateMany(
  { category: "Electronics" },
  { $mul: { price: 1.10 } }           // 10% price increase on all electronics
)

// ── $min / $max — Conditional update ──
db.products.updateOne(
  { name: "iPhone 15 Pro" },
  { $min: { price: 129900 } }         // Only update if 129900 < current price
)

db.users.updateOne(
  { name: "Ritesh Singh" },
  { $max: { highScore: 9500 } }       // Only update if 9500 > current highScore
)

// ── $rename — Rename a field ──
db.users.updateMany(
  {},
  { $rename: { "city": "location" } }  // Rename "city" to "location" for all users
)

// ── $currentDate — Set to current date ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  {
    $set: { status: "active" },
    $currentDate: {
      lastLogin: true,                 // Sets to Date
      "metadata.lastModified": { $type: "timestamp" }  // Sets to Timestamp
    }
  }
)
```

---

### 3.3 Array Update Operators — Modify Arrays in Place

```javascript
// ┌──────────────────────────────────────────────────────────────┐
// │  ARRAY UPDATE OPERATORS                                      │
// ├───────────┬──────────────────────────────────────────────────┤
// │ $push     │ Add element(s) to array                          │
// │ $pull     │ Remove element(s) matching condition              │
// │ $pop      │ Remove first (-1) or last (1) element            │
// │ $addToSet │ Add only if not already present (no duplicates)   │
// │ $each     │ Modifier for $push/$addToSet — add multiple      │
// │ $sort     │ Modifier — sort the array after $push            │
// │ $slice    │ Modifier — keep only N elements after $push      │
// │ $position │ Modifier — insert at specific position           │
// │ $pullAll  │ Remove all matching values from array            │
// └───────────┴──────────────────────────────────────────────────┘

// ── $push — Add to array ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $push: { skills: "MongoDB" } }
)
// skills: ["Python"] → ["Python", "MongoDB"]

// ── $push with $each — Add multiple elements ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $push: { skills: { $each: ["Docker", "AWS", "Redis"] } } }
)
// skills: ["Python", "MongoDB"] → ["Python", "MongoDB", "Docker", "AWS", "Redis"]

// ── $push with $sort and $slice — Maintain "Top N" list ──
db.products.updateOne(
  { name: "MacBook Pro 16" },
  {
    $push: {
      reviews: {
        $each: [{ user: "Frank", rating: 5, comment: "Incredible!" }],
        $sort: { rating: -1 },    // Sort reviews by rating descending
        $slice: 10                 // Keep only top 10 reviews
      }
    }
  }
)

// ── $push with $position — Insert at specific index ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $push: { skills: { $each: ["Rust"], $position: 0 } } }
)
// Adds "Rust" at the BEGINNING of the array

// ── $addToSet — Add only if unique ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $addToSet: { skills: "Python" } }     // Already exists → NO CHANGE
)
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $addToSet: { skills: "Terraform" } }   // New → ADDED
)

// ── $addToSet with $each ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $addToSet: { skills: { $each: ["Python", "Go", "Elixir"] } } }
)
// Only adds "Go" and "Elixir" (Python already exists)

// ── $pull — Remove matching elements ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $pull: { skills: "Redis" } }           // Remove "Redis" from array
)

// $pull with condition (array of objects):
db.products.updateOne(
  { name: "MacBook Pro 16" },
  { $pull: { reviews: { rating: { $lt: 4 } } } }
)
// Remove all reviews with rating < 4

// ── $pullAll — Remove specific values ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $pullAll: { skills: ["Docker", "AWS"] } }  // Remove both
)

// ── $pop — Remove first or last ──
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $pop: { skills: 1 } }      // Remove LAST element
)
db.users.updateOne(
  { name: "Ritesh Singh" },
  { $pop: { skills: -1 } }     // Remove FIRST element
)
```

### 3.4 Positional Operators — Update Specific Array Elements

```javascript
// ── $ (positional) — Update the FIRST matched array element ──
db.products.updateOne(
  { name: "MacBook Pro 16", "reviews.user": "Bob" },
  { $set: { "reviews.$.rating": 5 } }
)
// Updates Bob's rating from 4 to 5
// $ = "the element that matched the query"

// ── $[] (all positional) — Update ALL elements ──
db.products.updateOne(
  { name: "MacBook Pro 16" },
  { $inc: { "reviews.$[].rating": 1 } }
)
// Increment ALL review ratings by 1

// ── $[<identifier>] (filtered positional) — Update elements matching filter ──
db.products.updateOne(
  { name: "MacBook Pro 16" },
  { $set: { "reviews.$[elem].verified": true } },
  { arrayFilters: [{ "elem.rating": { $gte: 4 } }] }
)
// Set verified=true ONLY for reviews with rating >= 4
```

---

### 3.5 updateOne() vs updateMany() vs replaceOne()

```javascript
// ── updateOne() — First matching document ──
db.users.updateOne(
  { city: "Delhi" },
  { $set: { timezone: "IST" } }
)
// Updates only the FIRST user in Delhi

// ── updateMany() — ALL matching documents ──
db.users.updateMany(
  { city: "Delhi" },
  { $set: { timezone: "IST" } }
)
// Updates ALL users in Delhi

// ── replaceOne() — Replace ENTIRE document ──
db.users.replaceOne(
  { name: "Ritesh Singh" },
  {
    name: "Ritesh Singh",
    email: "ritesh.new@example.com",
    age: 29,
    city: "Noida",
    skills: ["MongoDB", "Python", "Node.js"],
    isActive: true
  }
)
// ⚠️ The old document is COMPLETELY replaced!
// Only _id is preserved. All other fields from old doc are GONE.
// This is different from $set (which only modifies specified fields)
```

---

### 3.6 Upsert — Update OR Insert

```javascript
// Upsert = Update if exists, Insert if doesn't

db.users.updateOne(
  { email: "newuser@example.com" },        // Filter
  {
    $set: { name: "New User", age: 22 },   // Update fields
    $setOnInsert: {                         // These ONLY apply on insert
      createdAt: new Date(),
      isActive: true,
      role: "user"
    }
  },
  { upsert: true }                         // ← Enable upsert!
)

// If email exists     → Updates name and age only
// If email NOT exists → Creates new document with ALL fields
//                        (email + $set fields + $setOnInsert fields)
```

```
When to use Upsert:

✅ Counters:       Track page views — increment if exists, create with 1 if not
✅ User profiles:  Create on first visit, update on subsequent visits
✅ Idempotent ops: Processing events — safe to retry without duplicates
✅ Configuration:  Set default config, override if exists
```

---

### 3.7 findOneAndUpdate() — Update & Return in One Shot

```javascript
// ── Return the document BEFORE update ──
const oldDoc = db.users.findOneAndUpdate(
  { name: "Ritesh Singh" },
  { $inc: { loginCount: 1 } },
  { returnDocument: "before" }     // Default: returns old version
)

// ── Return the document AFTER update ──
const newDoc = db.users.findOneAndUpdate(
  { name: "Ritesh Singh" },
  { $inc: { loginCount: 1 } },
  { returnDocument: "after" }      // Returns updated version
)

// ── With upsert + projection ──
db.inventory.findOneAndUpdate(
  { sku: "ABC123" },
  { $inc: { quantity: -1 } },
  {
    returnDocument: "after",
    upsert: true,
    projection: { sku: 1, quantity: 1, _id: 0 }
  }
)
```

> 💡 **Pro Tip**: `findOneAndUpdate()` is **atomic** — no other operation can happen between the find and update. This is perfect for:
> - **Counters**: `$inc` and get the new value
> - **Queues**: Find a task with status "pending" and set it to "processing"
> - **Inventory**: Decrement stock and check if it went below 0

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PART 4: DELETE Operations
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### 4.1 Delete Methods

```javascript
// ── deleteOne() — Delete FIRST matching document ──
db.users.deleteOne({ name: "Ravi Kumar" })
// Output: { acknowledged: true, deletedCount: 1 }

// ── deleteMany() — Delete ALL matching documents ──
db.users.deleteMany({ isActive: false })
// Deletes all inactive users

// ── Delete ALL documents (empty filter) ──
db.logs.deleteMany({})     // ⚠️ Deletes EVERYTHING in collection!
// But keeps the collection and its indexes

// ── Drop the entire collection ──
db.logs.drop()             // ⚠️ Removes collection + all indexes!
// Faster than deleteMany({}) for clearing everything

// ── findOneAndDelete() — Delete & return the document ──
const deleted = db.tasks.findOneAndDelete(
  { status: "completed" },
  { sort: { completedAt: 1 } }    // Delete the oldest completed task
)
// Returns the deleted document (useful for queue patterns)
```

### 4.2 Safe Delete Patterns

```javascript
// ── Always verify before bulk delete ──

// Step 1: COUNT what will be deleted
const count = db.orders.countDocuments({
  status: "cancelled",
  createdAt: { $lt: new Date("2023-01-01") }
})
print(`Will delete ${count} documents`)

// Step 2: PREVIEW a few
db.orders.find({
  status: "cancelled",
  createdAt: { $lt: new Date("2023-01-01") }
}).limit(5)

// Step 3: If confident, DELETE
db.orders.deleteMany({
  status: "cancelled",
  createdAt: { $lt: new Date("2023-01-01") }
})

// ── Soft Delete Pattern (recommended for important data) ──
// Instead of deleting, mark as deleted:
db.users.updateOne(
  { _id: userId },
  {
    $set: {
      isDeleted: true,
      deletedAt: new Date(),
      deletedBy: "admin"
    }
  }
)

// Then filter out deleted docs in all queries:
db.users.find({ isDeleted: { $ne: true } })
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## PART 5: BULK OPERATIONS
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### 5.1 Bulk Write — High Performance Batch Operations

```javascript
// When you have thousands of operations, don't do them one by one!
// bulkWrite() sends them in a SINGLE round trip.

db.products.bulkWrite([
  // Operation 1: Insert
  {
    insertOne: {
      document: { name: "AirPods Pro 2", price: 24900, stock: 500 }
    }
  },
  // Operation 2: Update
  {
    updateOne: {
      filter: { name: "iPhone 15 Pro" },
      update: { $inc: { stock: -1 } }
    }
  },
  // Operation 3: Update Many
  {
    updateMany: {
      filter: { category: "Electronics" },
      update: { $set: { onSale: true } }
    }
  },
  // Operation 4: Delete
  {
    deleteOne: {
      filter: { name: "Old Product" }
    }
  },
  // Operation 5: Replace
  {
    replaceOne: {
      filter: { name: "Widget" },
      replacement: { name: "Super Widget", price: 999, stock: 100 }
    }
  }
], { ordered: false })   // unordered = faster (parallelizable)

// Output:
// {
//   acknowledged: true,
//   insertedCount: 1,
//   matchedCount: ...,
//   modifiedCount: ...,
//   deletedCount: 1,
//   upsertedCount: 0
// }
```

```
Performance Comparison:

Individual operations (1000 inserts):
  → 1000 round trips to server
  → ~5 seconds

bulkWrite (1000 inserts):
  → 1 round trip (batched internally)
  → ~0.2 seconds

  🚀 25x faster!
```

---

## 🗺️ CRUD Cheat Sheet — The Complete Reference

```
┌────────────────────────────────────────────────────────────────────┐
│                    MongoDB CRUD CHEAT SHEET                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  CREATE:                                                           │
│    db.col.insertOne({...})                                        │
│    db.col.insertMany([{...}, {...}])                              │
│                                                                    │
│  READ:                                                             │
│    db.col.findOne({filter})                                       │
│    db.col.find({filter}, {projection})                            │
│    db.col.find().sort().skip().limit()                             │
│    db.col.countDocuments({filter})                                 │
│    db.col.distinct("field")                                       │
│                                                                    │
│  UPDATE:                                                           │
│    db.col.updateOne({filter}, {$set: {...}})                      │
│    db.col.updateMany({filter}, {$set: {...}})                     │
│    db.col.replaceOne({filter}, {newDoc})                          │
│    db.col.findOneAndUpdate({filter}, {update}, {options})         │
│                                                                    │
│  DELETE:                                                           │
│    db.col.deleteOne({filter})                                     │
│    db.col.deleteMany({filter})                                    │
│    db.col.findOneAndDelete({filter})                              │
│    db.col.drop()                                                  │
│                                                                    │
│  OPERATORS:                                                        │
│    Compare: $eq $ne $gt $gte $lt $lte $in $nin                    │
│    Logical: $and $or $not $nor                                    │
│    Element: $exists $type                                         │
│    Array:   $all $elemMatch $size                                 │
│    Update:  $set $unset $inc $mul $min $max $rename               │
│    Array:   $push $pull $pop $addToSet $pullAll                   │
│    Modify:  $each $sort $slice $position                          │
│    Pos:     $ $[] $[<id>] (with arrayFilters)                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│                   CHAPTER 3B.3 SUMMARY                        │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. insertOne/insertMany — ordered vs unordered matters       │
│                                                               │
│  2. find() returns a CURSOR — chain sort/skip/limit on it    │
│                                                               │
│  3. Comparison: $eq $gt $lt $in etc. (like SQL WHERE)         │
│                                                               │
│  4. $elemMatch is CRITICAL for array-of-objects queries       │
│                                                               │
│  5. Projections control which fields return (1=include)       │
│                                                               │
│  6. $set updates fields; replaceOne replaces the WHOLE doc   │
│                                                               │
│  7. Array operators ($push, $pull, $addToSet) are powerful   │
│                                                               │
│  8. Use $[identifier] + arrayFilters for targeted updates    │
│                                                               │
│  9. findOneAndUpdate = atomic find+update (queues/counters)  │
│                                                               │
│  10. bulkWrite() for batch ops = 25x faster than individual  │
│                                                               │
│  11. Soft delete > hard delete for important data             │
│                                                               │
│  12. skip() is slow for large offsets → use range-based       │
│      pagination with _id instead                              │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|-------------|-------|
| **[3B.4 — MongoDB Aggregation Framework](./04-MongoDB-Aggregation.md)** | The most powerful feature in MongoDB — pipeline-based data processing |

---

> **"CRUD is the language MongoDB speaks. Now you're fluent. Time to learn poetry — the Aggregation Framework."**
