# 🍃 Chapter 3B.4 — MongoDB Aggregation Framework

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~5–6 hours
> **Prerequisites:** Chapter 3B.3 (MongoDB CRUD)

> **"If CRUD is speaking MongoDB, Aggregation is writing poetry with it. This is where MongoDB truly shines."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the **pipeline architecture** — how stages flow into each other
- Master every major pipeline stage: `$match`, `$group`, `$project`, `$sort`, `$lookup`, `$unwind`, `$addFields`, `$facet`, and more
- Build **real-world aggregation pipelines** for analytics, reporting, and data transformation
- Know when to use Aggregation vs. simple `find()` queries
- Optimize pipelines for **performance**
- Think in pipelines like a **data engineer**

---

## 1. What is the Aggregation Framework?

### The SQL Comparison

In SQL, you combine `SELECT`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `JOIN`, and subqueries into a single statement. It works, but complex queries become **unreadable monsters**.

MongoDB took a different approach — **pipelines**:

```
SQL:
  SELECT department, AVG(salary) as avg_sal
  FROM employees
  WHERE status = 'active'
  GROUP BY department
  HAVING AVG(salary) > 50000
  ORDER BY avg_sal DESC

MongoDB Aggregation Pipeline:
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  $match  │───►│  $group  │───►│  $match  │───►│  $sort   │───►│  Output  │
  │status=   │    │by dept   │    │avg>50000 │    │desc      │    │          │
  │'active'  │    │AVG(sal)  │    │(HAVING)  │    │          │    │          │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
       ↑               ↑               ↑              ↑
    WHERE          GROUP BY          HAVING         ORDER BY
```

### Why Pipelines are Brilliant

```
Think of it like a FACTORY ASSEMBLY LINE:

  Raw Materials (Documents)
       │
       ▼
  ┌─────────────┐
  │ Station 1   │  Filter out defects        ($match)
  │ Quality     │
  └──────┬──────┘
         │ Only good parts pass through
         ▼
  ┌─────────────┐
  │ Station 2   │  Group by type             ($group)
  │ Sorting     │
  └──────┬──────┘
         │ Organized batches
         ▼
  ┌─────────────┐
  │ Station 3   │  Reshape the product       ($project)
  │ Assembly    │
  └──────┬──────┘
         │ Finished products
         ▼
  ┌─────────────┐
  │ Station 4   │  Package & label           ($sort, $limit)
  │ Packaging   │
  └──────┬──────┘
         │
         ▼
  📦 Final Output

  Each station:
  ✅ Receives documents from the previous station
  ✅ Transforms them
  ✅ Passes results to the next station
  ✅ Doesn't care about other stations
```

### Syntax

```javascript
db.collection.aggregate([
  { $stage1: { ... } },
  { $stage2: { ... } },
  { $stage3: { ... } },
  // ... as many stages as you need
])
```

---

## 2. Setup — Our Practice Data

```javascript
use ecommerce

// Insert sample orders data
db.orders.insertMany([
  {
    _id: 1,
    customer: "Ritesh",
    customerId: 101,
    items: [
      { product: "iPhone 15", price: 79999, qty: 1, category: "Electronics" },
      { product: "AirPods Pro", price: 24999, qty: 2, category: "Electronics" }
    ],
    total: 129997,
    status: "delivered",
    paymentMethod: "credit_card",
    city: "Delhi",
    orderDate: ISODate("2024-11-15"),
    deliveredDate: ISODate("2024-11-18")
  },
  {
    _id: 2,
    customer: "Priya",
    customerId: 102,
    items: [
      { product: "MacBook Pro", price: 199900, qty: 1, category: "Electronics" },
      { product: "Magic Mouse", price: 7500, qty: 1, category: "Accessories" }
    ],
    total: 207400,
    status: "delivered",
    paymentMethod: "upi",
    city: "Mumbai",
    orderDate: ISODate("2024-11-20"),
    deliveredDate: ISODate("2024-11-23")
  },
  {
    _id: 3,
    customer: "Amit",
    customerId: 103,
    items: [
      { product: "Samsung S24", price: 69999, qty: 1, category: "Electronics" }
    ],
    total: 69999,
    status: "shipped",
    paymentMethod: "debit_card",
    city: "Bangalore",
    orderDate: ISODate("2024-12-01")
  },
  {
    _id: 4,
    customer: "Sneha",
    customerId: 104,
    items: [
      { product: "Book: MongoDB Guide", price: 599, qty: 3, category: "Books" },
      { product: "Laptop Stand", price: 2499, qty: 1, category: "Accessories" }
    ],
    total: 4296,
    status: "delivered",
    paymentMethod: "upi",
    city: "Delhi",
    orderDate: ISODate("2024-10-05"),
    deliveredDate: ISODate("2024-10-08")
  },
  {
    _id: 5,
    customer: "Ritesh",
    customerId: 101,
    items: [
      { product: "iPad Air", price: 59900, qty: 1, category: "Electronics" },
      { product: "Apple Pencil", price: 11900, qty: 1, category: "Accessories" }
    ],
    total: 71800,
    status: "cancelled",
    paymentMethod: "credit_card",
    city: "Delhi",
    orderDate: ISODate("2024-12-10")
  },
  {
    _id: 6,
    customer: "Karan",
    customerId: 105,
    items: [
      { product: "PS5", price: 49999, qty: 1, category: "Gaming" },
      { product: "Controller", price: 5999, qty: 2, category: "Gaming" }
    ],
    total: 61997,
    status: "delivered",
    paymentMethod: "credit_card",
    city: "Pune",
    orderDate: ISODate("2024-11-25"),
    deliveredDate: ISODate("2024-11-28")
  },
  {
    _id: 7,
    customer: "Neha",
    customerId: 106,
    items: [
      { product: "Kindle Paperwhite", price: 13999, qty: 1, category: "Electronics" },
      { product: "Book: System Design", price: 799, qty: 2, category: "Books" }
    ],
    total: 15597,
    status: "processing",
    paymentMethod: "upi",
    city: "Chennai",
    orderDate: ISODate("2024-12-15")
  },
  {
    _id: 8,
    customer: "Priya",
    customerId: 102,
    items: [
      { product: "AirPods Max", price: 59900, qty: 1, category: "Electronics" }
    ],
    total: 59900,
    status: "delivered",
    paymentMethod: "credit_card",
    city: "Mumbai",
    orderDate: ISODate("2024-12-20"),
    deliveredDate: ISODate("2024-12-22")
  }
])

print("✅ Sample data inserted! Ready for aggregation.")
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## STAGE-BY-STAGE DEEP DIVE
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 3. $match — Filter Documents (The WHERE Clause)

> **Always put `$match` as EARLY as possible** — it reduces the number of documents flowing through the rest of the pipeline.

```javascript
// ── Basic $match ──
db.orders.aggregate([
  { $match: { status: "delivered" } }
])
// Returns only delivered orders (like WHERE status = 'delivered')

// ── $match with operators ──
db.orders.aggregate([
  { $match: {
    status: "delivered",
    total: { $gte: 50000 },
    orderDate: { $gte: ISODate("2024-11-01") }
  }}
])
// Delivered orders worth ≥₹50,000, placed after Nov 1, 2024

// ── $match with $or ──
db.orders.aggregate([
  { $match: {
    $or: [
      { status: "delivered" },
      { status: "shipped" }
    ]
  }}
])
// Or simpler with $in:
db.orders.aggregate([
  { $match: { status: { $in: ["delivered", "shipped"] } } }
])
```

```
Performance Tip:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  $match at the BEGINNING of a pipeline can use INDEXES!     │
  │                                                              │
  │  ✅ { $match: { status: "delivered" } }  ← FIRST stage      │
  │     → Uses index on "status" (if exists)                     │
  │                                                              │
  │  ❌ After $group or $project, $match CANNOT use indexes      │
  │     → Documents have been transformed, original fields gone  │
  │                                                              │
  │  Rule: Filter EARLY, transform LATE                          │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘
```

---

## 4. $group — Aggregate Data (The GROUP BY)

> This is the **heart** of the aggregation framework.

### 4.1 Basic $group

```javascript
// Syntax:
// { $group: { _id: <grouping expression>, <field>: { <accumulator>: <expr> } } }

// ── Count orders per status ──
db.orders.aggregate([
  { $group: {
    _id: "$status",              // Group by the "status" field
    count: { $sum: 1 }           // Count documents in each group
  }}
])
// Output:
// { _id: "delivered", count: 5 }
// { _id: "shipped", count: 1 }
// { _id: "cancelled", count: 1 }
// { _id: "processing", count: 1 }

// ── Total revenue per city ──
db.orders.aggregate([
  { $group: {
    _id: "$city",
    totalRevenue: { $sum: "$total" },
    orderCount: { $sum: 1 }
  }}
])
// Output:
// { _id: "Delhi", totalRevenue: 206093, orderCount: 3 }
// { _id: "Mumbai", totalRevenue: 267300, orderCount: 2 }
// ...

// ── Group by NULL (_id: null) = Aggregate ALL documents ──
db.orders.aggregate([
  { $group: {
    _id: null,                          // No grouping — treat all as one group
    totalOrders: { $sum: 1 },
    totalRevenue: { $sum: "$total" },
    avgOrderValue: { $avg: "$total" },
    maxOrder: { $max: "$total" },
    minOrder: { $min: "$total" }
  }}
])
// Output: Single document with totals across ALL orders
```

### 4.2 All Accumulator Operators

```javascript
// ┌──────────────────────────────────────────────────────────────┐
// │  GROUP ACCUMULATOR OPERATORS                                 │
// ├───────────┬──────────────────────────────────────────────────┤
// │ $sum      │ Sum of values (or count with $sum: 1)            │
// │ $avg      │ Average of values                                │
// │ $min      │ Minimum value                                    │
// │ $max      │ Maximum value                                    │
// │ $first    │ First value in group (respects $sort before)     │
// │ $last     │ Last value in group                              │
// │ $push     │ Push values into an array                        │
// │ $addToSet │ Push UNIQUE values into an array                 │
// │ $count    │ Count of documents (MongoDB 5.0+)                │
// │ $stdDevPop│ Standard deviation (population)                  │
// │ $stdDevSamp│ Standard deviation (sample)                     │
// │ $mergeObjects│ Merge documents into one                      │
// └───────────┴──────────────────────────────────────────────────┘

// ── $push — Collect values into an array ──
db.orders.aggregate([
  { $group: {
    _id: "$city",
    customers: { $push: "$customer" }    // All customer names in this city
  }}
])
// { _id: "Delhi", customers: ["Ritesh", "Sneha", "Ritesh"] }
// Note: Ritesh appears twice (two orders from Delhi)

// ── $addToSet — Collect UNIQUE values ──
db.orders.aggregate([
  { $group: {
    _id: "$city",
    uniqueCustomers: { $addToSet: "$customer" }
  }}
])
// { _id: "Delhi", uniqueCustomers: ["Ritesh", "Sneha"] }  ← No duplicates!

// ── $first / $last — Useful after $sort ──
db.orders.aggregate([
  { $sort: { total: -1 } },             // Sort by total descending
  { $group: {
    _id: "$city",
    highestOrderValue: { $first: "$total" },   // Highest order per city
    highestOrderCustomer: { $first: "$customer" }
  }}
])
```

### 4.3 Grouping by Multiple Fields

```javascript
// ── Group by city AND status ──
db.orders.aggregate([
  { $group: {
    _id: {
      city: "$city",
      status: "$status"
    },
    count: { $sum: 1 },
    totalRevenue: { $sum: "$total" }
  }}
])
// Output:
// { _id: { city: "Delhi", status: "delivered" }, count: 1, totalRevenue: 4296 }
// { _id: { city: "Delhi", status: "cancelled" }, count: 1, totalRevenue: 71800 }
// { _id: { city: "Mumbai", status: "delivered" }, count: 2, totalRevenue: 267300 }
// ...

// ── Group by month (extract from date) ──
db.orders.aggregate([
  { $group: {
    _id: {
      year: { $year: "$orderDate" },
      month: { $month: "$orderDate" }
    },
    orderCount: { $sum: 1 },
    revenue: { $sum: "$total" }
  }},
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

---

## 5. $project — Reshape Documents (The SELECT)

> `$project` is like SQL's `SELECT` — choose which fields to include, rename them, compute new fields.

```javascript
// ── Include/Exclude fields ──
db.orders.aggregate([
  { $project: {
    _id: 0,                           // Exclude _id
    customer: 1,                       // Include
    total: 1,                          // Include
    status: 1                          // Include
  }}
])

// ── Rename fields ──
db.orders.aggregate([
  { $project: {
    customerName: "$customer",          // Rename: customer → customerName
    orderTotal: "$total",               // Rename: total → orderTotal
    orderStatus: "$status"
  }}
])

// ── Computed fields ──
db.orders.aggregate([
  { $project: {
    customer: 1,
    total: 1,
    // Tax calculation (18% GST)
    tax: { $multiply: ["$total", 0.18] },
    // Total with tax
    grandTotal: { $multiply: ["$total", 1.18] },
    // Number of items in this order
    itemCount: { $size: "$items" },
    // Year from date
    orderYear: { $year: "$orderDate" },
    // Month name
    orderMonth: { $month: "$orderDate" },
    // Full date formatted
    formattedDate: {
      $dateToString: { format: "%d-%m-%Y", date: "$orderDate" }
    }
  }}
])

// ── String operations ──
db.orders.aggregate([
  { $project: {
    customerUpper: { $toUpper: "$customer" },
    customerLower: { $toLower: "$customer" },
    nameLength: { $strLenCP: "$customer" },
    greeting: { $concat: ["Hello, ", "$customer", "!"] },
    initial: { $substrCP: ["$customer", 0, 1] }
  }}
])

// ── Conditional fields ($cond — like CASE WHEN) ──
db.orders.aggregate([
  { $project: {
    customer: 1,
    total: 1,
    orderCategory: {
      $cond: {
        if: { $gte: ["$total", 100000] },
        then: "Premium",
        else: {
          $cond: {
            if: { $gte: ["$total", 50000] },
            then: "Standard",
            else: "Budget"
          }
        }
      }
    }
  }}
])

// ── Cleaner alternative: $switch (like CASE WHEN) ──
db.orders.aggregate([
  { $project: {
    customer: 1,
    total: 1,
    orderCategory: {
      $switch: {
        branches: [
          { case: { $gte: ["$total", 100000] }, then: "Premium" },
          { case: { $gte: ["$total", 50000] },  then: "Standard" },
          { case: { $gte: ["$total", 10000] },  then: "Regular" }
        ],
        default: "Budget"
      }
    }
  }}
])
```

---

## 6. $addFields / $set — Add Fields Without Dropping Others

> `$project` replaces the document shape. `$addFields` (alias: `$set`) **adds to** the existing document.

```javascript
// ── $addFields — Add new fields, keep everything else ──
db.orders.aggregate([
  { $addFields: {
    tax: { $multiply: ["$total", 0.18] },
    itemCount: { $size: "$items" },
    isHighValue: { $gte: ["$total", 50000] }
  }}
])
// Original fields + 3 new fields!

// ── $set is just an ALIAS for $addFields (MongoDB 4.2+) ──
db.orders.aggregate([
  { $set: {
    deliveryDays: {
      $cond: {
        if: { $and: [
          { $ne: ["$deliveredDate", null] },
          { $ifNull: ["$deliveredDate", false] }
        ]},
        then: {
          $dateDiff: {
            startDate: "$orderDate",
            endDate: "$deliveredDate",
            unit: "day"
          }
        },
        else: null
      }
    }
  }}
])
```

> 💡 **Pro Tip**: Use `$addFields`/`$set` when you want to **enrich** documents without specifying every single field. Use `$project` when you want to **reshape** the output.

---

## 7. $sort — Order Results

```javascript
// ── Sort ascending ──
db.orders.aggregate([
  { $sort: { total: 1 } }              // Cheapest first
])

// ── Sort descending ──
db.orders.aggregate([
  { $sort: { total: -1 } }             // Most expensive first
])

// ── Multi-field sort ──
db.orders.aggregate([
  { $sort: { city: 1, total: -1 } }    // City A→Z, then highest total first
])

// ── Sort AFTER $group (common pattern) ──
db.orders.aggregate([
  { $group: {
    _id: "$city",
    totalRevenue: { $sum: "$total" }
  }},
  { $sort: { totalRevenue: -1 } }      // Cities by revenue, highest first
])
```

```
Performance Note:

  $sort at the BEGINNING of pipeline:
  ✅ Can use indexes (if index exists on the sort field)
  
  $sort AFTER $group or $project:
  ❌ Cannot use indexes (documents have been transformed)
  ❌ Must sort in memory
  ⚠️  MongoDB has a 100MB memory limit for sort!
     Use { allowDiskUse: true } for larger sorts:
  
  db.orders.aggregate([...], { allowDiskUse: true })
```

---

## 8. $limit and $skip — Pagination in Aggregation

```javascript
// ── Top 3 most expensive orders ──
db.orders.aggregate([
  { $sort: { total: -1 } },
  { $limit: 3 }
])

// ── Pagination (Page 2, 5 items per page) ──
db.orders.aggregate([
  { $sort: { orderDate: -1 } },
  { $skip: 5 },                         // Skip first page (5 items)
  { $limit: 5 }                         // Take 5 items
])
```

---

## 9. $unwind — Explode Arrays into Individual Documents

> `$unwind` takes an array field and creates a **separate document for each element**.

```javascript
// Before $unwind:
// { _id: 1, customer: "Ritesh", items: [
//   { product: "iPhone", price: 79999 },
//   { product: "AirPods", price: 24999 }
// ]}

db.orders.aggregate([
  { $match: { _id: 1 } },
  { $unwind: "$items" }
])

// After $unwind (2 documents from 1):
// { _id: 1, customer: "Ritesh", items: { product: "iPhone", price: 79999 } }
// { _id: 1, customer: "Ritesh", items: { product: "AirPods", price: 24999 } }
```

```
Visual:
                         $unwind
  ┌──────────────────┐              ┌──────────────────┐
  │ order: 1         │              │ order: 1         │
  │ items: [         │              │ item: "iPhone"   │
  │   "iPhone",      │  ────────►   ├──────────────────┤
  │   "AirPods",     │              │ order: 1         │
  │   "Magic Mouse"  │              │ item: "AirPods"  │
  │ ]                │              ├──────────────────┤
  └──────────────────┘              │ order: 1         │
                                    │ item: "Magic..." │
  1 document                        └──────────────────┘
                                    3 documents
```

### 9.1 Real-World $unwind Use Cases

```javascript
// ── Revenue by PRODUCT (not by order) ──
db.orders.aggregate([
  { $unwind: "$items" },                      // Explode items array
  { $group: {
    _id: "$items.product",
    totalRevenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
    totalQuantity: { $sum: "$items.qty" }
  }},
  { $sort: { totalRevenue: -1 } }
])
// Now we can see revenue per individual product!

// ── Revenue by CATEGORY ──
db.orders.aggregate([
  { $unwind: "$items" },
  { $group: {
    _id: "$items.category",
    revenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
    itemsSold: { $sum: "$items.qty" }
  }},
  { $sort: { revenue: -1 } }
])
// Output:
// { _id: "Electronics", revenue: 563596, itemsSold: 7 }
// { _id: "Gaming", revenue: 61997, itemsSold: 3 }
// { _id: "Accessories", revenue: 21899, itemsSold: 3 }
// { _id: "Books", revenue: 3395, itemsSold: 5 }

// ── Handle empty arrays with preserveNullAndEmptyArrays ──
db.orders.aggregate([
  { $unwind: {
    path: "$items",
    preserveNullAndEmptyArrays: true    // Keep docs with empty/missing arrays
  }}
])
// Without this option: documents with empty "items" array are DROPPED
// With this option: they're kept (items field will be null/missing)
```

> 💡 **Pro Tip**: `$unwind` + `$group` is the **bread and butter** of analytics. Explode arrays, then re-aggregate however you want.

---

## 10. $lookup — JOIN Collections (The LEFT JOIN)

> `$lookup` is MongoDB's way of joining data from different collections. It performs a **LEFT OUTER JOIN**.

### 10.1 Basic $lookup

```javascript
// First, let's create a customers collection:
db.customers.insertMany([
  { _id: 101, name: "Ritesh", email: "ritesh@example.com", tier: "Gold" },
  { _id: 102, name: "Priya", email: "priya@example.com", tier: "Platinum" },
  { _id: 103, name: "Amit", email: "amit@example.com", tier: "Silver" },
  { _id: 104, name: "Sneha", email: "sneha@example.com", tier: "Silver" },
  { _id: 105, name: "Karan", email: "karan@example.com", tier: "Gold" },
  { _id: 106, name: "Neha", email: "neha@example.com", tier: "Bronze" }
])
```

```javascript
// ── Join orders with customers ──
db.orders.aggregate([
  { $lookup: {
    from: "customers",             // The collection to join with
    localField: "customerId",      // Field in orders
    foreignField: "_id",           // Field in customers
    as: "customerDetails"          // Output array field name
  }}
])

// Output (each order now has customer info):
// {
//   _id: 1,
//   customer: "Ritesh",
//   customerId: 101,
//   total: 129997,
//   ...
//   customerDetails: [                    ← Array (usually 1 element)
//     { _id: 101, name: "Ritesh", email: "ritesh@example.com", tier: "Gold" }
//   ]
// }
```

```
Visual:

  ORDERS collection              CUSTOMERS collection
  ┌──────────────────┐          ┌──────────────────┐
  │ _id: 1           │          │ _id: 101         │
  │ customerId: 101──┼──$lookup─┼→name: "Ritesh"   │
  │ total: 129997    │          │ tier: "Gold"      │
  └──────────────────┘          └──────────────────┘
  
  Result:
  ┌──────────────────────────────────────┐
  │ _id: 1                               │
  │ customerId: 101                      │
  │ total: 129997                        │
  │ customerDetails: [{                  │
  │   _id: 101, name: "Ritesh",         │
  │   tier: "Gold"                       │
  │ }]                                   │
  └──────────────────────────────────────┘
```

### 10.2 $lookup + $unwind — Flatten the Join

```javascript
// $lookup returns an ARRAY. Usually you want a single object.
// Use $unwind to flatten it:

db.orders.aggregate([
  { $lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customerInfo"
  }},
  { $unwind: "$customerInfo" },          // Array → single object
  { $project: {
    _id: 0,
    orderId: "$_id",
    customer: "$customerInfo.name",
    email: "$customerInfo.email",
    tier: "$customerInfo.tier",
    total: 1,
    status: 1,
    orderDate: 1
  }}
])
```

### 10.3 Advanced $lookup with Pipeline

```javascript
// MongoDB 3.6+ supports a PIPELINE inside $lookup
// This gives you full aggregation power inside the join!

db.orders.aggregate([
  { $lookup: {
    from: "customers",
    let: { custId: "$customerId" },       // Variables from outer document
    pipeline: [                            // Sub-pipeline on the joined collection
      { $match: {
        $expr: { $eq: ["$_id", "$$custId"] }   // Join condition
      }},
      { $project: {                        // Only return specific fields
        _id: 0,
        name: 1,
        tier: 1,
        email: 1
      }}
    ],
    as: "customerInfo"
  }},
  { $unwind: "$customerInfo" }
])

// Why use pipeline $lookup?
// ✅ Filter the joined collection BEFORE joining (performance!)
// ✅ Project only needed fields (less data transfer)
// ✅ Add computed fields in the sub-pipeline
// ✅ Join on multiple conditions
// ✅ Uncorrelated sub-queries (no let needed)
```

> ⚠️ **Performance Warning**: `$lookup` performs the join **in the aggregation pipeline, not in the storage engine**. For very large collections, this can be slow. If you frequently join the same data, consider **embedding** the data instead (denormalization).

---

## 11. $count — Count the Results

```javascript
// ── Count delivered orders ──
db.orders.aggregate([
  { $match: { status: "delivered" } },
  { $count: "deliveredOrders" }
])
// Output: { deliveredOrders: 5 }

// This is equivalent to:
db.orders.countDocuments({ status: "delivered" })
// But $count can be used AFTER other pipeline stages
```

---

## 12. $bucket and $bucketAuto — Histogram/Range Grouping

```javascript
// ── $bucket — Group into custom ranges ──
db.orders.aggregate([
  { $bucket: {
    groupBy: "$total",                  // Field to bucket
    boundaries: [0, 10000, 50000, 100000, 500000],  // Range boundaries
    default: "Other",                   // For values outside boundaries
    output: {
      count: { $sum: 1 },
      orders: { $push: "$customer" },
      avgTotal: { $avg: "$total" }
    }
  }}
])
// Output:
// { _id: 0,      count: 1, ... }     ← Orders ₹0 - ₹9,999
// { _id: 10000,  count: 2, ... }     ← Orders ₹10,000 - ₹49,999
// { _id: 50000,  count: 3, ... }     ← Orders ₹50,000 - ₹99,999
// { _id: 100000, count: 2, ... }     ← Orders ₹100,000 - ₹499,999

// ── $bucketAuto — Let MongoDB decide the ranges ──
db.orders.aggregate([
  { $bucketAuto: {
    groupBy: "$total",
    buckets: 4,                         // "Give me 4 evenly distributed groups"
    output: {
      count: { $sum: 1 },
      avgTotal: { $avg: "$total" }
    }
  }}
])
```

---

## 13. $facet — Multiple Pipelines in One Query

> `$facet` runs **multiple aggregation pipelines in parallel** on the same input and returns all results in a single document. Like running multiple queries at once.

```javascript
// ── Dashboard data in ONE query ──
db.orders.aggregate([
  { $facet: {
    
    // Pipeline 1: Order stats by status
    "statusBreakdown": [
      { $group: {
        _id: "$status",
        count: { $sum: 1 },
        revenue: { $sum: "$total" }
      }},
      { $sort: { revenue: -1 } }
    ],
    
    // Pipeline 2: Top 3 customers by spending
    "topCustomers": [
      { $group: {
        _id: "$customer",
        totalSpent: { $sum: "$total" },
        orderCount: { $sum: 1 }
      }},
      { $sort: { totalSpent: -1 } },
      { $limit: 3 }
    ],
    
    // Pipeline 3: Monthly trend
    "monthlyTrend": [
      { $group: {
        _id: { $month: "$orderDate" },
        revenue: { $sum: "$total" },
        orders: { $sum: 1 }
      }},
      { $sort: { _id: 1 } }
    ],
    
    // Pipeline 4: Total summary
    "summary": [
      { $group: {
        _id: null,
        totalRevenue: { $sum: "$total" },
        totalOrders: { $sum: 1 },
        avgOrderValue: { $avg: "$total" }
      }}
    ]
  }}
])

// Output: ONE document with 4 arrays:
// {
//   statusBreakdown: [...],
//   topCustomers: [...],
//   monthlyTrend: [...],
//   summary: [...]
// }
```

> 💡 **Pro Tip**: `$facet` is a **dashboard builder's best friend**. One database call → all the data for your dashboard. Each facet pipeline runs on the same input documents, so the data is always consistent.

---

## 14. $out and $merge — Write Results to a Collection

```javascript
// ── $out — Write results to a NEW collection (replaces if exists!) ──
db.orders.aggregate([
  { $group: {
    _id: "$city",
    totalRevenue: { $sum: "$total" },
    orderCount: { $sum: 1 }
  }},
  { $out: "city_revenue_summary" }      // ⚠️ Replaces entire collection!
])

// Now query the materialized results:
db.city_revenue_summary.find()

// ── $merge (MongoDB 4.2+) — More flexible, can UPDATE existing docs ──
db.orders.aggregate([
  { $group: {
    _id: "$city",
    totalRevenue: { $sum: "$total" },
    orderCount: { $sum: 1 },
    lastUpdated: { $max: new Date() }
  }},
  { $merge: {
    into: "city_stats",                 // Target collection
    on: "_id",                          // Match field
    whenMatched: "merge",               // Update existing docs
    whenNotMatched: "insert"            // Insert new docs
  }}
])
// $merge does NOT drop the collection — it intelligently updates it!
```

```
$out vs $merge:

  ┌──────────────┬────────────────────┬──────────────────────┐
  │              │  $out              │  $merge              │
  ├──────────────┼────────────────────┼──────────────────────┤
  │ Behavior     │ REPLACES entire    │ UPSERTS into         │
  │              │ collection         │ existing collection  │
  │ Same DB?     │ Must be same DB    │ Any DB (even diff)   │
  │ Existing     │ Dropped & recreated│ Updated/Merged       │
  │ collection   │                    │                      │
  │ Use Case     │ Rebuild reports    │ Incremental updates  │
  │ Safety       │ ⚠️ Destructive     │ ✅ Non-destructive   │
  └──────────────┴────────────────────┴──────────────────────┘
```

---

## 15. Useful Expression Operators Reference

### 15.1 Date Operators

```javascript
db.orders.aggregate([
  { $project: {
    year:       { $year: "$orderDate" },
    month:      { $month: "$orderDate" },
    day:        { $dayOfMonth: "$orderDate" },
    dayOfWeek:  { $dayOfWeek: "$orderDate" },      // 1=Sun, 7=Sat
    dayOfYear:  { $dayOfYear: "$orderDate" },
    hour:       { $hour: "$orderDate" },
    formatted:  { $dateToString: {
      format: "%Y-%m-%d %H:%M",
      date: "$orderDate",
      timezone: "Asia/Kolkata"
    }},
    // Date arithmetic (MongoDB 5.0+)
    daysAgo: {
      $dateDiff: {
        startDate: "$orderDate",
        endDate: new Date(),
        unit: "day"
      }
    }
  }}
])
```

### 15.2 Math Operators

```javascript
// $add, $subtract, $multiply, $divide, $mod
// $abs, $ceil, $floor, $round, $sqrt, $pow, $log, $ln

db.orders.aggregate([
  { $project: {
    total: 1,
    gst: { $round: [{ $multiply: ["$total", 0.18] }, 2] },
    discount10pct: { $round: [{ $multiply: ["$total", 0.10] }, 0] },
    afterDiscount: { $subtract: [
      "$total",
      { $multiply: ["$total", 0.10] }
    ]}
  }}
])
```

### 15.3 Conditional Operators

```javascript
// $cond (if-then-else)
// $switch (multi-branch case)
// $ifNull (coalesce)

db.orders.aggregate([
  { $project: {
    customer: 1,
    // $ifNull — provide default for missing fields
    deliveredDate: { $ifNull: ["$deliveredDate", "Not yet delivered"] },
    
    // $cond — ternary
    paymentType: {
      $cond: {
        if: { $eq: ["$paymentMethod", "credit_card"] },
        then: "Credit",
        else: "Other"
      }
    }
  }}
])
```

---

## 16. Real-World Aggregation Pipelines

### 16.1 E-Commerce Dashboard

```javascript
// "Show me a complete sales dashboard"
db.orders.aggregate([
  // Step 1: Only delivered orders
  { $match: { status: "delivered" } },
  
  // Step 2: Get customer details
  { $lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customerInfo"
  }},
  { $unwind: "$customerInfo" },
  
  // Step 3: Explode items for per-product analysis
  { $unwind: "$items" },
  
  // Step 4: Group by product category
  { $group: {
    _id: "$items.category",
    totalRevenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
    itemsSold: { $sum: "$items.qty" },
    uniqueCustomers: { $addToSet: "$customer" },
    avgItemPrice: { $avg: "$items.price" }
  }},
  
  // Step 5: Add computed fields
  { $addFields: {
    revenueFormatted: {
      $concat: ["₹", { $toString: "$totalRevenue" }]
    },
    customerCount: { $size: "$uniqueCustomers" }
  }},
  
  // Step 6: Sort by revenue
  { $sort: { totalRevenue: -1 } },
  
  // Step 7: Clean up output
  { $project: {
    category: "$_id",
    _id: 0,
    totalRevenue: 1,
    revenueFormatted: 1,
    itemsSold: 1,
    customerCount: 1,
    avgItemPrice: { $round: ["$avgItemPrice", 0] }
  }}
])
```

### 16.2 Customer Lifetime Value (CLV)

```javascript
// "Calculate how much each customer has spent total, their order frequency,
//  average order value, and classify them into tiers"

db.orders.aggregate([
  { $match: { status: { $ne: "cancelled" } } },
  
  { $group: {
    _id: "$customerId",
    customerName: { $first: "$customer" },
    totalSpent: { $sum: "$total" },
    orderCount: { $sum: 1 },
    avgOrderValue: { $avg: "$total" },
    firstOrder: { $min: "$orderDate" },
    lastOrder: { $max: "$orderDate" },
    cities: { $addToSet: "$city" }
  }},
  
  { $addFields: {
    clvTier: {
      $switch: {
        branches: [
          { case: { $gte: ["$totalSpent", 200000] }, then: "💎 Diamond" },
          { case: { $gte: ["$totalSpent", 100000] }, then: "🥇 Gold" },
          { case: { $gte: ["$totalSpent", 50000] },  then: "🥈 Silver" }
        ],
        default: "🥉 Bronze"
      }
    },
    daysSinceLastOrder: {
      $dateDiff: {
        startDate: "$lastOrder",
        endDate: new Date(),
        unit: "day"
      }
    }
  }},
  
  { $sort: { totalSpent: -1 } },
  
  { $project: {
    _id: 0,
    customerId: "$_id",
    customerName: 1,
    clvTier: 1,
    totalSpent: 1,
    orderCount: 1,
    avgOrderValue: { $round: ["$avgOrderValue", 0] },
    daysSinceLastOrder: 1
  }}
])
```

### 16.3 Time-Based Trend Analysis

```javascript
// "Show me daily order trends with running totals"

db.orders.aggregate([
  { $group: {
    _id: {
      $dateToString: { format: "%Y-%m-%d", date: "$orderDate" }
    },
    dailyRevenue: { $sum: "$total" },
    orderCount: { $sum: 1 }
  }},
  
  { $sort: { _id: 1 } },
  
  // Running total using $setWindowFields (MongoDB 5.0+)
  { $setWindowFields: {
    sortBy: { _id: 1 },
    output: {
      cumulativeRevenue: {
        $sum: "$dailyRevenue",
        window: { documents: ["unbounded", "current"] }
      },
      cumulativeOrders: {
        $sum: "$orderCount",
        window: { documents: ["unbounded", "current"] }
      },
      movingAvgRevenue: {
        $avg: "$dailyRevenue",
        window: { documents: [-2, 0] }    // 3-day moving average
      }
    }
  }},
  
  { $project: {
    date: "$_id",
    _id: 0,
    dailyRevenue: 1,
    orderCount: 1,
    cumulativeRevenue: 1,
    movingAvgRevenue: { $round: ["$movingAvgRevenue", 0] }
  }}
])
```

---

## 17. Aggregation Pipeline Optimization

### 17.1 Pipeline Optimization Rules

```
┌──────────────────────────────────────────────────────────────────┐
│              AGGREGATION OPTIMIZATION RULES                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. $match FIRST — Filter as early as possible                    │
│     → Reduces documents for all subsequent stages                 │
│     → Can use indexes (only when first!)                          │
│                                                                   │
│  2. $project EARLY — Remove unnecessary fields                    │
│     → Less data in memory for remaining stages                   │
│     → Especially important before $group and $sort               │
│                                                                   │
│  3. $sort + $limit together — MongoDB optimizes this              │
│     → Only keeps top N in memory (not full sort)                  │
│                                                                   │
│  4. $match after $group → moves before $group if possible         │
│     → MongoDB's query planner does this automatically             │
│                                                                   │
│  5. Adjacent $match stages → merged into one                      │
│     → MongoDB combines them automatically                         │
│                                                                   │
│  6. Avoid $unwind when possible                                   │
│     → Use array operators ($filter, $reduce, $map) instead       │
│     → $unwind creates more documents = more work                  │
│                                                                   │
│  7. Use { allowDiskUse: true } for large datasets                 │
│     → Default 100MB RAM limit per stage                          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 17.2 Bad vs Good Pipeline

```javascript
// ❌ BAD — Processes ALL documents, then filters
db.orders.aggregate([
  { $lookup: { from: "customers", ... } },      // JOIN on all 1M orders
  { $unwind: "$items" },                         // Explode all items
  { $group: { _id: "$city", ... } },             // Group everything
  { $match: { _id: "Delhi" } }                   // THEN filter... too late!
])

// ✅ GOOD — Filter first, then process the small set
db.orders.aggregate([
  { $match: { city: "Delhi" } },                 // Filter FIRST → 100 docs
  { $lookup: { from: "customers", ... } },       // JOIN on 100 docs
  { $unwind: "$items" },                         // Explode only Delhi items
  { $group: { _id: "$items.category", ... } }    // Group the small set
])

// The good version can be 100x faster!
```

### 17.3 Check Pipeline Performance with explain()

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "delivered" } },
  { $group: { _id: "$city", total: { $sum: "$total" } } },
  { $sort: { total: -1 } }
])

// Look for:
// - "stage": "IXSCAN" (good!) vs "COLLSCAN" (bad — full scan)
// - "nReturned" vs "totalDocsExamined" (ratio should be close)
// - "executionTimeMillis" (lower = better)
```

---

## 18. Aggregation vs find() — When to Use What

```
┌──────────────────────────────────────────────────────────────────┐
│  USE find() WHEN:                  USE aggregate() WHEN:         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ✅ Simple queries                  ✅ GROUP BY / aggregations     │
│  ✅ Filter + sort + limit          ✅ JOINs ($lookup)             │
│  ✅ Basic projections              ✅ Computed/derived fields      │
│  ✅ Count documents                ✅ Array transformations        │
│  ✅ Single collection              ✅ Multiple collections         │
│  ✅ CRUD operations                ✅ Reshape documents            │
│                                    ✅ Faceted search              │
│                                    ✅ Analytics & reporting        │
│                                    ✅ Window functions             │
│                                    ✅ Write results ($out/$merge) │
│                                                                   │
│  find() is SIMPLER and FASTER     aggregate() is POWERFUL but     │
│  for basic reads.                  more complex.                  │
│                                                                   │
│  Rule: If find() can do it, use find(). Otherwise, aggregate().  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🗺️ Pipeline Stages Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│            AGGREGATION PIPELINE STAGES REFERENCE                  │
├─────────────────┬────────────────────────────────────────────────┤
│  STAGE          │  PURPOSE                       │  SQL EQUIV.   │
├─────────────────┼────────────────────────────────┼───────────────┤
│  $match         │  Filter documents              │  WHERE/HAVING │
│  $group         │  Group & aggregate             │  GROUP BY     │
│  $project       │  Reshape (include/exclude)     │  SELECT       │
│  $addFields     │  Add fields (keep others)      │  SELECT + *   │
│  $sort          │  Order results                 │  ORDER BY     │
│  $limit         │  Limit results                 │  LIMIT/TOP    │
│  $skip          │  Skip results                  │  OFFSET       │
│  $unwind        │  Explode arrays                │  LATERAL JOIN │
│  $lookup        │  Join collections              │  LEFT JOIN    │
│  $count         │  Count documents               │  COUNT(*)     │
│  $bucket        │  Range-based grouping          │  CASE+GROUP BY│
│  $facet         │  Parallel pipelines            │  UNION ALL    │
│  $out           │  Write to collection           │  INTO TABLE   │
│  $merge         │  Upsert into collection        │  MERGE INTO   │
│  $replaceRoot   │  Replace document root         │  —            │
│  $sample        │  Random sample                 │  ORDER BY RND │
│  $redact        │  Field-level access control    │  —            │
│  $setWindowFields│ Window functions (5.0+)       │  OVER()       │
│  $unionWith     │  Union with another collection │  UNION ALL    │
│  $graphLookup   │  Recursive graph traversal     │  RECURSIVE CTE│
│  $densify       │  Fill gaps in time series      │  —            │
│  $fill          │  Fill null/missing values      │  COALESCE     │
└─────────────────┴────────────────────────────────┴───────────────┘
```

---

## 🧠 Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│                   CHAPTER 3B.4 SUMMARY                        │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Aggregation = pipeline of stages, each transforming data │
│                                                               │
│  2. $match FIRST → uses indexes, reduces data early          │
│                                                               │
│  3. $group + accumulators ($sum, $avg, $push) = GROUP BY     │
│                                                               │
│  4. $project reshapes; $addFields enriches without dropping  │
│                                                               │
│  5. $unwind explodes arrays → combine with $group for        │
│     per-element analytics                                     │
│                                                               │
│  6. $lookup = LEFT JOIN between collections                   │
│     (pipeline version is more powerful)                       │
│                                                               │
│  7. $facet = multiple pipelines in one call (dashboards!)     │
│                                                               │
│  8. $bucket = histogram-style grouping                       │
│                                                               │
│  9. $merge > $out (non-destructive, incremental updates)     │
│                                                               │
│  10. Optimize: $match early, $project early, avoid           │
│      unnecessary $unwind, use explain()                      │
│                                                               │
│  11. {allowDiskUse: true} for large sorts (>100MB)           │
│                                                               │
│  12. $setWindowFields (5.0+) = SQL window functions          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|-------------|-------|
| **[3B.5 — MongoDB Indexing & Performance](./05-MongoDB-Indexing.md)** | Make your queries blazing fast with the right indexes |

---

> **"The aggregation framework is MongoDB's most powerful feature. If you can think in pipelines, you can answer any data question. You're no longer just querying — you're engineering data."**
