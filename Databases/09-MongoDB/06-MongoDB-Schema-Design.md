# 🏗️ Chapter 3B.6 — MongoDB Schema Design Patterns

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 3B.3 (CRUD), Chapter 3B.4 (Aggregation), Chapter 3B.5 (Indexing)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the **fundamental** embedding vs referencing decision
- Apply **12 battle-tested schema design patterns** used at Netflix, eBay, and Uber
- Know the **anti-patterns** that have crashed production systems
- Design schemas that are **read-optimized** (because reads dominate 90%+ of workloads)
- Think in **documents**, not tables — the mindset shift that changes everything

---

## 🧠 The #1 Rule of MongoDB Schema Design

> **"Design your schema based on how your APPLICATION reads and writes data — NOT based on how the data relates."**

In SQL, you normalize first and think about queries later.
In MongoDB, **you start with queries and design the schema to serve them**.

```
    SQL Mindset:                       MongoDB Mindset:
    ──────────────────                 ──────────────────────────
    "What are my entities?"            "What does my app need to display?"
    "How do they relate?"              "What queries will I run?"
    "Normalize everything"             "What data is accessed together?"
    "JOIN at query time"               "Embed what's read together"
    "One fact in one place"            "Acceptable duplication for speed"
```

---

## 🤔 The Great Decision: Embed vs Reference

This is the **single most important** decision in MongoDB schema design.

### Option A: Embedding (Denormalization)

```javascript
// Embedding: Put related data INSIDE the document
{
  _id: ObjectId("..."),
  name: "John Doe",
  email: "john@example.com",
  address: {                         // ← Embedded subdocument
    street: "123 Main St",
    city: "New York",
    state: "NY",
    zip: "10001"
  },
  orders: [                          // ← Embedded array
    { orderId: 1, product: "Laptop", amount: 1200, date: ISODate("2024-01-15") },
    { orderId: 2, product: "Mouse",  amount: 25,   date: ISODate("2024-01-20") }
  ]
}

// ✅ Single read gets everything — ONE disk seek
// ✅ Atomic update (update user + order in one operation)
// ✅ No $lookup (JOIN) needed — blazing fast
```

### Option B: Referencing (Normalization)

```javascript
// User document
{
  _id: ObjectId("user123"),
  name: "John Doe",
  email: "john@example.com",
  addressId: ObjectId("addr456")     // ← Reference to address
}

// Address document (separate collection)
{
  _id: ObjectId("addr456"),
  street: "123 Main St",
  city: "New York",
  state: "NY",
  zip: "10001"
}

// Order documents (separate collection)
{
  _id: ObjectId("order789"),
  userId: ObjectId("user123"),       // ← Reference back to user
  product: "Laptop",
  amount: 1200,
  date: ISODate("2024-01-15")
}

// ❌ Multiple reads needed (user + address + orders)
// ❌ Need $lookup to JOIN
// ✅ But no data duplication
// ✅ Large sub-datasets don't bloat the parent
```

### The Decision Framework

```
    ┌──────────────────────────────────────────────────────────────────┐
    │                    EMBED vs REFERENCE                            │
    │                                                                  │
    │                    ASK THESE QUESTIONS:                           │
    │                                                                  │
    │  1. Is the data ALWAYS accessed together?                        │
    │     YES → Embed  |  NO → Reference                              │
    │                                                                  │
    │  2. What's the cardinality?                                      │
    │     One-to-Few (1:3) → Embed                                    │
    │     One-to-Many (1:1000) → Embed or Reference (depends on size) │
    │     One-to-Millions (1:∞) → Reference ALWAYS                    │
    │                                                                  │
    │  3. Does the embedded data grow unboundedly?                     │
    │     YES → Reference  (arrays can't grow forever!)                │
    │     NO  → Embed                                                  │
    │                                                                  │
    │  4. Does the data need independent access?                       │
    │     YES → Reference  (if you query orders alone, separate them)  │
    │     NO  → Embed                                                  │
    │                                                                  │
    │  5. How often does the embedded data change?                     │
    │     Rarely → Embed   (user's name, birth date)                   │
    │     Often  → Reference (user's order status)                     │
    │                                                                  │
    │  6. Will the document exceed 16MB?                               │
    │     Risk of yes → Reference                                      │
    └──────────────────────────────────────────────────────────────────┘
```

### Cardinality Guide — The Numbers

```
    ┌─────────────────────┬───────────┬──────────────────────────────┐
    │ Cardinality         │ Strategy  │ Example                      │
    ├─────────────────────┼───────────┼──────────────────────────────┤
    │ One-to-One (1:1)    │ Embed     │ User → Profile               │
    │ One-to-Few (1:~5)   │ Embed     │ User → Addresses             │
    │ One-to-Many (1:100s)│ Embed     │ Blog Post → Comments (few)   │
    │ One-to-Many (1:1000s│ Reference │ User → Orders (thousands)    │
    │ One-to-Squillions   │ Reference │ Host → Log entries (millions)│
    │ Many-to-Many (N:M)  │ Reference │ Students ↔ Courses           │
    └─────────────────────┴───────────┴──────────────────────────────┘
```

---

## 📦 Pattern 1: Attribute Pattern

> **Problem:** You have many similar fields that are queried the same way, but field names vary.

```javascript
// ❌ BAD: Different products have different attributes
{
  title: "Heavy Duty Drill",
  // Clothing might have: size, color, material
  // Electronics might have: voltage, wattage, battery
  // Furniture might have: width, height, weight, material
  voltage: "220V",
  wattage: 1200,
  rpm: 3000,
  weight: "4.5 kg"
}
// Problem: Need separate indexes for EVERY attribute field!
// db.products.createIndex({ voltage: 1 })
// db.products.createIndex({ wattage: 1 })
// db.products.createIndex({ rpm: 1 })
// db.products.createIndex({ weight: 1 })   ... 100 more indexes?!

// ✅ GOOD: Attribute pattern — normalize the varying fields
{
  title: "Heavy Duty Drill",
  attributes: [
    { k: "voltage",  v: "220V",   unit: "volts" },
    { k: "wattage",  v: 1200,     unit: "watts" },
    { k: "rpm",      v: 3000,     unit: "rpm" },
    { k: "weight",   v: 4.5,      unit: "kg" }
  ]
}

// ONE index handles all attribute queries!
db.products.createIndex({ "attributes.k": 1, "attributes.v": 1 })

// Query: Find all products with wattage > 1000
db.products.find({ 
  attributes: { $elemMatch: { k: "wattage", v: { $gt: 1000 } } }
})
```

### When to Use

```
✅ Products with varying specifications (e-commerce)
✅ Movie/book metadata (different genres have different fields)
✅ IoT sensor data (different sensor types)
✅ Any polymorphic data with queryable attributes
```

---

## 📦 Pattern 2: Bucket Pattern

> **Problem:** You're storing massive amounts of time-series or sequential data — one document per event creates **millions** of tiny documents.

```javascript
// ❌ BAD: One document per sensor reading (too many documents!)
{ sensorId: "S001", temp: 22.5, timestamp: ISODate("2024-01-15T10:00:00Z") }
{ sensorId: "S001", temp: 22.7, timestamp: ISODate("2024-01-15T10:01:00Z") }
{ sensorId: "S001", temp: 22.3, timestamp: ISODate("2024-01-15T10:02:00Z") }
// ... 1 million documents per sensor per day!
// Massive index size, horrible query performance

// ✅ GOOD: Bucket pattern — group readings into time buckets
{
  sensorId: "S001",
  date: ISODate("2024-01-15"),              // Bucket: 1 day
  hour: 10,                                  // Bucket: 1 hour
  nReadings: 60,                             // Count of readings in bucket
  readings: [                                // Array of readings
    { temp: 22.5, ts: ISODate("2024-01-15T10:00:00Z") },
    { temp: 22.7, ts: ISODate("2024-01-15T10:01:00Z") },
    { temp: 22.3, ts: ISODate("2024-01-15T10:02:00Z") },
    // ... 60 readings per hour
  ],
  // Pre-aggregated stats (Computed pattern bonus!)
  stats: {
    min: 22.1,
    max: 23.4,
    avg: 22.6,
    sum: 1356.0
  }
}

// Benefits:
// 📉 60x fewer documents (1 per hour instead of 1 per minute)
// 📉 60x smaller index
// 📈 Pre-computed stats = instant analytics
// 📈 Reads fetch 1 document instead of 60
```

### Choosing Bucket Size

```
    ┌────────────────────────────────────┬───────────────────────────────┐
    │ Bucket too small                   │ Bucket too large              │
    │ (1 minute of data)                 │ (1 month of data)             │
    ├────────────────────────────────────┼───────────────────────────────┤
    │ Too many documents (like no bucket)│ Documents too large (>16MB)   │
    │ Wasted overhead per document       │ Slow to update                │
    │                                    │ Can't fit in RAM              │
    └────────────────────────────────────┴───────────────────────────────┘
    
    Rule of thumb: Bucket size should create documents ~200KB each.
    
    Common bucket sizes:
    • IoT sensors reading every second → 1 hour per bucket
    • Stock prices every second → 1 minute per bucket
    • Web analytics per page → 1 day per bucket
    • Weather data per city → 1 day per bucket
```

---

## 📦 Pattern 3: Outlier Pattern

> **Problem:** Most documents are small, but a few **outliers** are massive. The outliers ruin your schema.

```javascript
// Example: Book sales tracking
// 99.9% of books sell < 1000 copies → embed buyers list is fine
// But "Harry Potter" sells 500 million copies → array would EXPLODE

// ✅ Solution: Normal case embedded, outlier overflows to separate collection

// Normal book (99.9% of cases)
{
  _id: "book_normal",
  title: "Introduction to Databases",
  buyers: ["user1", "user2", "user3", ... "user800"],
  hasOverflow: false
}

// Outlier book (0.1% — mega bestseller)
{
  _id: "book_hp",
  title: "Harry Potter",
  buyers: ["user1", "user2", ... "user1000"],  // Only first 1000
  hasOverflow: true                             // ← Flag!
}

// Overflow collection (only for outliers)
{
  bookId: "book_hp",
  page: 2,
  buyers: ["user1001", "user1002", ... "user2000"]
}
{
  bookId: "book_hp",
  page: 3,
  buyers: ["user2001", "user2002", ... "user3000"]
}

// Query logic:
function getBuyers(bookId) {
  const book = db.books.findOne({ _id: bookId });
  let allBuyers = book.buyers;
  
  if (book.hasOverflow) {
    const overflow = db.bookBuyersOverflow.find({ bookId }).toArray();
    overflow.forEach(page => allBuyers = allBuyers.concat(page.buyers));
  }
  
  return allBuyers;
}

// ✅ 99.9% of queries: single document read (fast!)
// ✅ 0.1% of queries: extra reads (acceptable for outliers)
```

### When to Use

```
✅ Social media: Most users have 200 followers, celebrities have 200 million
✅ E-commerce: Most products have 10 reviews, viral products have 100,000
✅ Analytics: Most pages get 100 views/day, homepage gets 10 million
✅ Any "long tail" distribution where few entities are orders of magnitude larger
```

---

## 📦 Pattern 4: Subset Pattern

> **Problem:** Documents are large, but you usually only need a **small portion** of the data.

```javascript
// Product page shows: name, price, and the 10 MOST RECENT reviews
// But the product has 50,000 reviews total!

// ❌ BAD: All reviews embedded (document too large, slow to load)
{
  title: "MacBook Pro",
  price: 2499,
  reviews: [ /* 50,000 reviews = ~5MB document! */ ]
}

// ✅ GOOD: Subset pattern — keep the most useful subset embedded
{
  _id: "macbook_pro",
  title: "MacBook Pro",
  price: 2499,
  reviewCount: 50000,
  avgRating: 4.7,
  recentReviews: [                    // ← Only the 10 most recent
    { user: "Alice", rating: 5, text: "Amazing!", date: ISODate("2024-03-15") },
    { user: "Bob",   rating: 4, text: "Great but expensive", date: ISODate("2024-03-14") },
    // ... 8 more
  ]
}

// Separate collection for ALL reviews
{
  productId: "macbook_pro",
  user: "Charlie",
  rating: 5,
  text: "Best laptop ever",
  date: ISODate("2023-06-20")
}

// Product page load: 1 document read (fast!) ⚡
// "Show all reviews" button: paginated query to reviews collection (lazy load)
```

### Common Subset Applications

```
✅ E-commerce: Product → 10 most recent reviews (full reviews in separate collection)
✅ Social media: User profile → 20 most recent posts (all posts in posts collection)
✅ News: Article → 5 most popular comments (all comments separately)
✅ Messaging: Chat → last 50 messages (message history in separate collection)
```

---

## 📦 Pattern 5: Computed Pattern

> **Problem:** Computing aggregations on-the-fly is **expensive**. Why compute the same thing 1 million times?

```javascript
// ❌ BAD: Compute total/average every time someone views the product page
// This aggregation runs on EVERY page view:
db.reviews.aggregate([
  { $match: { productId: "macbook_pro" } },
  { $group: { _id: null, avg: { $avg: "$rating" }, count: { $sum: 1 } } }
])
// If product page gets 100K views/day → 100K aggregations! 🐌

// ✅ GOOD: Pre-compute and store the result
{
  _id: "macbook_pro",
  title: "MacBook Pro",
  price: 2499,
  // Pre-computed values (updated on each new review)
  computed: {
    totalReviews: 50000,
    avgRating: 4.7,
    ratingDistribution: {
      "5": 30000,
      "4": 12000,
      "3": 5000,
      "2": 2000,
      "1": 1000
    },
    totalSales: 125000,
    lastReviewDate: ISODate("2024-03-15"),
    weeklyViews: 45000
  }
}

// When a new review comes in:
db.products.updateOne(
  { _id: "macbook_pro" },
  {
    $inc: { 
      "computed.totalReviews": 1,
      "computed.ratingDistribution.5": 1   // If it's a 5-star review
    },
    $set: { "computed.lastReviewDate": new Date() },
    // Recalculate average (or use a running average formula)
  }
)

// Page load: 1 document read, zero aggregation! ⚡
```

### Running Average Formula

```javascript
// Instead of re-aggregating everything, compute incrementally:

function updateAverage(currentAvg, currentCount, newRating) {
  return ((currentAvg * currentCount) + newRating) / (currentCount + 1);
}

// When adding a new 5-star review:
// currentAvg = 4.7, currentCount = 50000, newRating = 5
// newAvg = (4.7 * 50000 + 5) / 50001 = 4.700006 ≈ 4.7

db.products.updateOne(
  { _id: "macbook_pro" },
  [
    { $set: {
        "computed.avgRating": {
          $divide: [
            { $add: [
              { $multiply: ["$computed.avgRating", "$computed.totalReviews"] },
              5  // new rating
            ]},
            { $add: ["$computed.totalReviews", 1] }
          ]
        },
        "computed.totalReviews": { $add: ["$computed.totalReviews", 1] }
      }
    }
  ]
)
```

---

## 📦 Pattern 6: Extended Reference Pattern

> **Problem:** You reference another collection but **always need a few fields** from it. The `$lookup` (JOIN) is killing performance.

```javascript
// ❌ BAD: Reference only — need $lookup every time
// Order document:
{ _id: "order1", customerId: ObjectId("cust123"), product: "Laptop", amount: 1200 }

// To display order with customer name:
db.orders.aggregate([
  { $lookup: { from: "customers", localField: "customerId", foreignField: "_id", as: "customer" } },
  { $unwind: "$customer" },
  { $project: { product: 1, amount: 1, customerName: "$customer.name" } }
])
// Expensive JOIN on every read! 🐌

// ✅ GOOD: Extended Reference — copy the fields you always need
{
  _id: "order1",
  customerId: ObjectId("cust123"),
  customerInfo: {                    // ← Copied from customer collection
    name: "John Doe",
    email: "john@example.com"
    // Only copy what you NEED — not the entire customer document!
  },
  product: "Laptop",
  amount: 1200
}

// Now the order page loads with ONE read — no $lookup! ⚡
// Trade-off: If customer changes name, you might need to update orders
// But how often does someone change their name? Almost never!
```

### When It's OK to Have Stale Copies

```
    ┌─────────────────────────────────────────────────────────────────┐
    │  "But what about data consistency?!" — The Staleness Spectrum   │
    │                                                                 │
    │  Rarely changes (SAFE to copy):                                 │
    │    • Person's name                                              │
    │    • Product title                                              │
    │    • Country/city name                                          │
    │    • Movie title, book ISBN                                     │
    │                                                                 │
    │  Sometimes changes (copy WITH update strategy):                 │
    │    • Product price → update periodically or on change event     │
    │    • User profile picture URL                                   │
    │    • Company name                                               │
    │                                                                 │
    │  Frequently changes (DON'T copy — use $lookup):                 │
    │    • Stock price                                                │
    │    • Live scores                                                │
    │    • Inventory count                                            │
    │    • User's current location                                    │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 📦 Pattern 7: Approximation Pattern

> **Problem:** Exact counts and stats are expensive. Sometimes **"close enough" is good enough**.

```javascript
// ❌ BAD: Update view count on EVERY page view
// 10 million views/day = 10 million writes to the same document!
db.articles.updateOne(
  { _id: "article1" },
  { $inc: { views: 1 } }
)
// Write amplification + lock contention = 🔥

// ✅ GOOD: Approximate — update every Nth view
function trackView(articleId) {
  // Only write 1 out of every 10 views (10% sampling)
  if (Math.random() < 0.1) {
    db.articles.updateOne(
      { _id: articleId },
      { $inc: { views: 10 } }  // Increment by 10 (compensate for sampling)
    );
  }
}

// 10M views → 1M writes (10x reduction!)
// Accuracy: ±3% (statistically) — good enough for "1.2M views"!

// More sophisticated: batch updates
const viewBuffer = {};
function trackViewBuffered(articleId) {
  viewBuffer[articleId] = (viewBuffer[articleId] || 0) + 1;
}

// Flush every 60 seconds
setInterval(() => {
  const bulk = db.articles.initializeUnorderedBulkOp();
  for (const [id, count] of Object.entries(viewBuffer)) {
    bulk.find({ _id: id }).updateOne({ $inc: { views: count } });
  }
  bulk.execute();
  viewBuffer = {};
}, 60000);
```

---

## 📦 Pattern 8: Schema Versioning Pattern

> **Problem:** Your schema evolves over time, but old documents still exist in the old format.

```javascript
// Version 1 (year 1): Simple address
{
  _id: "user1",
  schema_version: 1,
  name: "John Doe",
  address: "123 Main St, New York, NY 10001"
}

// Version 2 (year 2): Structured address
{
  _id: "user2",
  schema_version: 2,
  name: "Jane Doe",
  address: {
    street: "456 Oak Ave",
    city: "Boston",
    state: "MA",
    zip: "02101"
  }
}

// Version 3 (year 3): Multiple addresses
{
  _id: "user3",
  schema_version: 3,
  name: "Bob Smith",
  addresses: [
    { type: "home", street: "789 Pine Rd", city: "Chicago", state: "IL", zip: "60601" },
    { type: "work", street: "100 Corp Blvd", city: "Chicago", state: "IL", zip: "60602" }
  ]
}

// Application code handles all versions:
function getAddress(user) {
  switch (user.schema_version) {
    case 1:
      return parseAddressString(user.address);  // Parse "123 Main St, ..."
    case 2:
      return user.address;
    case 3:
      return user.addresses.find(a => a.type === "home") || user.addresses[0];
  }
}

// Migration strategy:
// Option A: Lazy migration — update documents when they're next accessed
// Option B: Background migration — batch update old documents over time
// Option C: Live with multiple versions — handle in application code
```

### Lazy Migration Example

```javascript
// Upgrade to version 3 when a user document is read
async function getUser(userId) {
  const user = await db.users.findOne({ _id: userId });
  
  if (user.schema_version < 3) {
    const migrated = migrateToV3(user);
    await db.users.replaceOne({ _id: userId }, migrated);
    return migrated;
  }
  
  return user;
}

function migrateToV3(user) {
  if (user.schema_version === 1) {
    const parsed = parseAddressString(user.address);
    return {
      ...user,
      schema_version: 3,
      addresses: [{ type: "home", ...parsed }]
    };
  }
  if (user.schema_version === 2) {
    return {
      ...user,
      schema_version: 3,
      addresses: [{ type: "home", ...user.address }]
    };
  }
}
```

---

## 📦 Pattern 9: Tree Pattern (for Hierarchies)

> **Problem:** Representing hierarchical data (categories, org charts, file systems) in MongoDB.

### Approach A: Parent Reference

```javascript
// Each node stores a reference to its parent
{ _id: "Electronics", parent: null }
{ _id: "Computers",   parent: "Electronics" }
{ _id: "Laptops",     parent: "Computers" }
{ _id: "Desktops",    parent: "Computers" }
{ _id: "Phones",      parent: "Electronics" }

// ✅ Find parent: easy
db.categories.findOne({ _id: "Laptops" })  // → parent: "Computers"

// ❌ Find all ancestors: requires recursive queries!
// ❌ Find all descendants: requires recursive queries!
```

### Approach B: Child Reference

```javascript
{ _id: "Electronics", children: ["Computers", "Phones"] }
{ _id: "Computers",   children: ["Laptops", "Desktops"] }
{ _id: "Laptops",     children: [] }

// ✅ Find children: easy
// ❌ Find ancestors: hard
```

### Approach C: Ancestors Array (Materialized Path)

```javascript
{ _id: "Electronics", ancestors: [],                                  path: "/Electronics" }
{ _id: "Computers",   ancestors: ["Electronics"],                     path: "/Electronics/Computers" }
{ _id: "Laptops",     ancestors: ["Electronics", "Computers"],        path: "/Electronics/Computers/Laptops" }
{ _id: "Gaming",      ancestors: ["Electronics", "Computers", "Laptops"], path: "/Electronics/Computers/Laptops/Gaming" }

// ✅ Find ALL ancestors of "Laptops": instant!
db.categories.find({ _id: { $in: ["Electronics", "Computers"] } })

// ✅ Find ALL descendants of "Electronics": instant!
db.categories.find({ ancestors: "Electronics" })

// ✅ Find path: just read the "path" field

// Index for fast descendant queries:
db.categories.createIndex({ ancestors: 1 })
```

### Approach D: Materialized Path (String Path)

```javascript
{ _id: "Laptops", path: ",Electronics,Computers,Laptops," }

// Find all descendants of Computers:
db.categories.find({ path: /,Computers,/ })

// Find all ancestors of Laptops:
// Parse the path string: ",Electronics,Computers,Laptops,"
```

### Which Tree Approach to Use?

```
┌──────────────────┬──────────┬───────────┬────────────┬──────────────┐
│ Operation        │ Parent   │ Child     │ Ancestors  │ Mat. Path    │
│                  │ Ref      │ Ref       │ Array      │ (String)     │
├──────────────────┼──────────┼───────────┼────────────┼──────────────┤
│ Find parent      │ ✅ Easy  │ 🟡 Query │ ✅ Easy    │ ✅ Parse     │
│ Find children    │ 🟡 Query│ ✅ Easy   │ 🟡 Query  │ 🟡 Regex    │
│ Find ancestors   │ 🔴 N+1  │ 🔴 N+1   │ ✅ Instant │ ✅ Parse     │
│ Find descendants │ 🔴 N+1  │ 🔴 N+1   │ ✅ Instant │ ✅ Regex     │
│ Move subtree     │ ✅ Easy  │ ✅ Easy   │ 🔴 Update │ 🔴 Update   │
│                  │          │           │   many     │   many       │
├──────────────────┼──────────┼───────────┼────────────┼──────────────┤
│ Best for         │ Simple   │ Simple    │ Read-heavy │ Read-heavy   │
│                  │ shallow  │ shallow   │ deep trees │ deep trees   │
└──────────────────┴──────────┴───────────┴────────────┴──────────────┘
```

---

## 📦 Pattern 10: Polymorphic Pattern

> **Problem:** A single collection stores documents of **different types** that share some common fields.

```javascript
// Single "vehicles" collection with different types

// Car
{
  type: "car",
  make: "Toyota",
  model: "Camry",
  year: 2024,
  doors: 4,
  trunkSize: "15 cubic ft",
  fuelType: "hybrid"
}

// Motorcycle
{
  type: "motorcycle",
  make: "Harley-Davidson",
  model: "Sportster",
  year: 2024,
  engineCC: 1200,
  hasSidecar: false
}

// Truck
{
  type: "truck",
  make: "Ford",
  model: "F-150",
  year: 2024,
  payload: "3300 lbs",
  towCapacity: "14000 lbs",
  bedLength: "6.5 ft"
}

// Common queries work across all types:
db.vehicles.find({ make: "Toyota", year: { $gte: 2020 } })

// Type-specific queries:
db.vehicles.find({ type: "truck", towCapacity: { $gt: "10000 lbs" } })

// Index strategy: Common fields + wildcard for type-specific
db.vehicles.createIndex({ type: 1, make: 1, year: 1 })     // Common queries
db.vehicles.createIndex({ "engineCC": 1 }, { partialFilterExpression: { type: "motorcycle" } })
```

### When to Use

```
✅ E-commerce: Single "products" collection (electronics, clothing, furniture)
✅ CMS: Single "content" collection (articles, videos, podcasts)
✅ Insurance: Single "policies" collection (auto, home, life)
✅ Social media: Single "feed" collection (posts, shares, ads)
```

---

## 📦 Pattern 11: Document Versioning Pattern

> **Problem:** You need to keep **history** of document changes (audit trail, undo, compliance).

```javascript
// Current state: always in main collection
{
  _id: "doc1",
  version: 5,                        // Current version
  title: "MongoDB Schema Design",
  content: "Latest content here...",
  lastModified: ISODate("2024-03-15"),
  modifiedBy: "alice"
}

// History: separate collection stores old versions
// Collection: document_history
{
  docId: "doc1",
  version: 4,
  title: "MongoDB Schema Design",
  content: "Previous content...",
  modifiedAt: ISODate("2024-03-10"),
  modifiedBy: "bob"
}
{
  docId: "doc1",
  version: 3,
  title: "MongoDB Schema Patterns",   // Title was different in v3!
  content: "Even older content...",
  modifiedAt: ISODate("2024-02-28"),
  modifiedBy: "alice"
}

// On update: save current to history, then update main
async function updateDocument(docId, newData, userId) {
  const current = await db.documents.findOne({ _id: docId });
  
  // 1. Archive current version
  await db.documentHistory.insertOne({
    docId: current._id,
    version: current.version,
    ...current,
    _id: undefined  // Let MongoDB generate new _id for history
  });
  
  // 2. Update to new version
  await db.documents.updateOne(
    { _id: docId },
    { 
      $set: { ...newData, modifiedBy: userId, lastModified: new Date() },
      $inc: { version: 1 }
    }
  );
}

// Get version history
db.documentHistory.find({ docId: "doc1" }).sort({ version: -1 })

// Restore to specific version
async function restoreVersion(docId, targetVersion) {
  const old = await db.documentHistory.findOne({ docId, version: targetVersion });
  await updateDocument(docId, old, "system_restore");
}
```

---

## 📦 Pattern 12: Pre-allocation Pattern

> **Problem:** Known future structure — allocate it now to avoid document growth and **moves**.

```javascript
// Reservation system: seats for a movie theater
// Pre-allocate all seats when a showing is created

{
  _id: "showing_2024_03_15_1900",
  movie: "Dune Part 3",
  theater: "Theater 5",
  showtime: ISODate("2024-03-15T19:00:00Z"),
  seats: {
    "A1": { status: "available", price: 15 },
    "A2": { status: "available", price: 15 },
    "A3": { status: "booked", price: 15, bookedBy: "user123" },
    "A4": { status: "available", price: 15 },
    // ... all seats pre-allocated
    "J20": { status: "available", price: 10 }
  },
  stats: {
    totalSeats: 200,
    booked: 45,
    available: 155
  }
}

// Book a seat — atomic operation!
db.showings.updateOne(
  { _id: "showing_2024_03_15_1900", "seats.B5.status": "available" },
  { 
    $set: { "seats.B5.status": "booked", "seats.B5.bookedBy": "user456" },
    $inc: { "stats.booked": 1, "stats.available": -1 }
  }
)
// If someone already booked B5, the filter won't match → no race condition!
```

---

## 🚨 Schema Design Anti-Patterns

```
    ┌────────────────────────────────────────────────────────────────────┐
    │  ❌ Anti-Pattern 1: Massive Arrays (Unbounded Growth)             │
    │     { user: "john", followers: ["user1", ... "user_5_million"] }  │
    │     → Document exceeds 16MB limit!                                │
    │     → Fix: Reference, Outlier pattern, or Bucket pattern          │
    ├────────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 2: Deep Nesting (>3-4 levels)                    │
    │     { a: { b: { c: { d: { e: { f: "value" } } } } } }           │
    │     → Hard to query, hard to index, hard to update                │
    │     → Fix: Flatten the structure                                  │
    ├────────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 3: Unnecessary Normalization                     │
    │     Splitting data into 10 collections like in SQL                │
    │     → 10 $lookups per query = death of performance                │
    │     → Fix: Embed what belongs together                            │
    ├────────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 4: Storing Join Data as "foreign keys"           │
    │     { orderId: 1, customerId: 5, productId: 42, shipperId: 3 }   │
    │     → This is a relational schema in document clothing            │
    │     → Fix: Embed or use Extended Reference pattern                │
    ├────────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 5: Using Collection Names as Data                │
    │     Collections: "orders_2023", "orders_2024", "orders_2025"      │
    │     → App code must know all collection names                     │
    │     → Fix: Single "orders" collection with date field + index     │
    ├────────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 6: Bloated Documents                             │
    │     Embedding EVERYTHING into one massive document                 │
    │     → Every read loads everything into memory                     │
    │     → Fix: Subset pattern, separate rarely-accessed data          │
    ├────────────────────────────────────────────────────────────────────┤
    │  ❌ Anti-Pattern 7: No Schema Validation                          │
    │     "MongoDB is schemaless so I don't need validation"            │
    │     → Garbage data will enter your database                       │
    │     → Fix: Use JSON Schema validation:                            │
    │       db.createCollection("users", { validator: { $jsonSchema: {  │
    │         required: ["name", "email"],                              │
    │         properties: { email: { bsonType: "string" } }             │
    │       }}})                                                        │
    └────────────────────────────────────────────────────────────────────┘
```

---

## 🏢 Real-World Case Study: E-Commerce Schema

Let's design a complete e-commerce schema using multiple patterns:

```javascript
// ═══════════════════════════════════════════════════════════
// PRODUCTS COLLECTION
// Patterns: Polymorphic, Attribute, Subset, Computed
// ═══════════════════════════════════════════════════════════
{
  _id: ObjectId("..."),
  sku: "LAPTOP-001",
  type: "electronics",                         // Polymorphic
  title: "MacBook Pro 16-inch",
  slug: "macbook-pro-16-inch",
  brand: "Apple",
  price: { amount: 2499, currency: "USD" },
  
  attributes: [                                // Attribute pattern
    { k: "processor", v: "M3 Pro" },
    { k: "ram", v: "18GB" },
    { k: "storage", v: "512GB SSD" },
    { k: "screen", v: "16.2 inch" }
  ],
  
  images: [
    { url: "/img/macbook-1.jpg", alt: "Front view", isPrimary: true },
    { url: "/img/macbook-2.jpg", alt: "Side view" }
  ],
  
  categories: ["Electronics", "Computers", "Laptops"],  // For breadcrumb
  
  computed: {                                  // Computed pattern
    avgRating: 4.7,
    totalReviews: 8542,
    totalSold: 125000,
    inStock: true,
    stockCount: 342
  },
  
  recentReviews: [                             // Subset pattern
    { user: "Alice", rating: 5, text: "Best laptop ever!", date: ISODate("2024-03-15") },
    // ... top 5 reviews
  ]
}

// ═══════════════════════════════════════════════════════════
// ORDERS COLLECTION
// Patterns: Extended Reference, Computed
// ═══════════════════════════════════════════════════════════
{
  _id: ObjectId("..."),
  orderNumber: "ORD-2024-00001",
  status: "shipped",
  
  customer: {                                  // Extended Reference
    _id: ObjectId("cust123"),
    name: "John Doe",
    email: "john@example.com"
  },
  
  items: [
    {
      productId: ObjectId("..."),
      sku: "LAPTOP-001",
      title: "MacBook Pro 16-inch",            // Extended Reference
      price: 2499,
      quantity: 1,
      subtotal: 2499
    },
    {
      productId: ObjectId("..."),
      sku: "CASE-042",
      title: "Laptop Sleeve",
      price: 49,
      quantity: 2,
      subtotal: 98
    }
  ],
  
  totals: {                                    // Computed pattern
    subtotal: 2597,
    tax: 233.73,
    shipping: 0,
    total: 2830.73
  },
  
  shipping: {
    address: { street: "123 Main St", city: "New York", state: "NY", zip: "10001" },
    method: "express",
    trackingNumber: "1Z999AA10123456784",
    estimatedDelivery: ISODate("2024-03-18")
  },
  
  timeline: [
    { status: "placed",    date: ISODate("2024-03-14T10:30:00Z") },
    { status: "paid",      date: ISODate("2024-03-14T10:31:00Z") },
    { status: "shipped",   date: ISODate("2024-03-15T08:00:00Z") }
  ],
  
  createdAt: ISODate("2024-03-14T10:30:00Z"),
  updatedAt: ISODate("2024-03-15T08:00:00Z")
}

// ═══════════════════════════════════════════════════════════
// USERS COLLECTION
// ═══════════════════════════════════════════════════════════
{
  _id: ObjectId("cust123"),
  email: "john@example.com",
  name: "John Doe",
  
  addresses: [                                 // Embed (One-to-Few)
    { type: "home", street: "123 Main St", city: "New York", state: "NY", zip: "10001", isDefault: true },
    { type: "work", street: "456 Corp Ave", city: "New York", state: "NY", zip: "10002" }
  ],
  
  paymentMethods: [                            // Embed (One-to-Few)
    { type: "card", last4: "4242", brand: "Visa", isDefault: true }
  ],
  
  computed: {
    totalOrders: 47,
    totalSpent: 12500.00,
    memberSince: ISODate("2020-01-15"),
    lastOrderDate: ISODate("2024-03-14")
  }
}

// ═══════════════════════════════════════════════════════════
// REVIEWS COLLECTION (separate — products use Subset pattern)
// ═══════════════════════════════════════════════════════════
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),
  userId: ObjectId("cust123"),
  userName: "John Doe",                        // Extended Reference
  rating: 5,
  title: "Best purchase ever",
  text: "Absolutely love this laptop...",
  helpful: { up: 42, down: 3 },
  verified: true,
  createdAt: ISODate("2024-03-15")
}
```

---

## 💡 Schema Design Decision Checklist

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │               SCHEMA DESIGN CHECKLIST                               │
    │                                                                     │
    │  Before writing any schema:                                         │
    │                                                                     │
    │  □ What are my top 5 most frequent queries?                        │
    │  □ What's the read/write ratio? (90/10? 50/50?)                    │
    │  □ What data is ALWAYS accessed together?                          │
    │  □ What's the cardinality of each relationship?                    │
    │  □ Will any array grow unboundedly?                                │
    │  □ Will any document exceed 16MB?                                  │
    │  □ What data changes frequently vs rarely?                         │
    │  □ Do I need atomic multi-field updates?                           │
    │  □ What indexes will I need? (Design schema to support them)       │
    │  □ Is JSON Schema validation in place?                             │
    │                                                                     │
    │  The answer to these questions DRIVES your schema.                  │
    │  Not the other way around.                                         │
    └─────────────────────────────────────────────────────────────────────┘
```

---

## 🧭 What's Next?

Your schema is designed. Now let's learn how to **scale it across multiple servers** and keep it **highly available**:

**Next Chapter → [3B.7 MongoDB Replication & Sharding](./07-MongoDB-Replication-Sharding.md)** 🔴🔥

> _"A great schema design on one server is good. A great schema design that scales to 100 servers is legendary."_

---

[← Previous: MongoDB Indexing & Performance](./05-MongoDB-Indexing.md) | [Index](../INDEX.md) | [Next: MongoDB Replication & Sharding →](./07-MongoDB-Replication-Sharding.md)
