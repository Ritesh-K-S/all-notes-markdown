# Chapter 26: Firestore & Datastore

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Firestore Fundamentals](#part-1-firestore-fundamentals)
- [Part 2: Native Mode vs Datastore Mode](#part-2-native-mode-vs-datastore-mode)
- [Part 3: Data Model — Documents & Collections](#part-3-data-model--documents--collections)
- [Part 4: Creating a Firestore Database](#part-4-creating-a-firestore-database)
- [Part 5: Reading & Writing Data](#part-5-reading--writing-data)
- [Part 6: Querying](#part-6-querying)
- [Part 7: Indexes](#part-7-indexes)
- [Part 8: Real-Time Listeners](#part-8-real-time-listeners)
- [Part 9: Security Rules (Native Mode)](#part-9-security-rules-native-mode)
- [Part 10: Transactions & Batched Writes](#part-10-transactions--batched-writes)
- [Part 11: Offline Support & Multi-Tab](#part-11-offline-support--multi-tab)
- [Part 12: Backups & Point-in-Time Recovery](#part-12-backups--point-in-time-recovery)
- [Part 13: Performance & Best Practices](#part-13-performance--best-practices)
- [Part 14: Terraform & CLI](#part-14-terraform--cli)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Quick Reference](#quick-reference)
- [Console Walkthrough: Managing & Deleting Firestore Data](#console-walkthrough-managing--deleting-firestore-data)
- [TTL Policies](#ttl-policies)
- [Firebase Emulator for Local Development](#firebase-emulator-for-local-development)
- [What's Next?](#whats-next)

---

## Overview

Firestore is Google Cloud's fully managed, serverless, NoSQL document database. It stores data as documents organized into collections — think of it like JSON objects grouped into folders. It has two modes: **Native mode** (the modern default with real-time sync and mobile SDKs) and **Datastore mode** (the legacy API for server-side workloads). Once you choose a mode for a project, you can't change it.

```
┌─────────────────────────────────────────────────────────────────────┐
│ WHAT YOU'LL LEARN                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ├── 1. What is Firestore and when to use it                        │
│ ├── 2. Native mode vs Datastore mode — which to pick              │
│ ├── 3. Data model: documents, collections, subcollections         │
│ ├── 4. Creating a database (console walkthrough)                  │
│ ├── 5. CRUD operations with client libraries                      │
│ ├── 6. Queries: simple, compound, collection groups               │
│ ├── 7. Indexes: single-field (auto) vs composite (manual)         │
│ ├── 8. Real-time listeners (snapshots)                            │
│ ├── 9. Security rules for client-side access                      │
│ ├── 10. Transactions and batched writes                           │
│ ├── 11. Offline support and multi-tab                             │
│ ├── 12. Backups and PITR                                          │
│ ├── 13. Performance tuning and best practices                     │
│ ├── 14. Terraform and gcloud CLI                                  │
│ └── 15. Real-world architecture patterns                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Firestore Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS FIRESTORE?                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Firestore is the next generation of Cloud Datastore. Google       │
│ merged the Datastore service into Firestore, giving it two modes. │
│                                                                       │
│ Key characteristics:                                                │
│ ├── NoSQL document database (not relational)                     │
│ ├── Serverless — no instances, clusters, or VMs to manage        │
│ ├── Automatic scaling — 0 to millions of ops/sec                 │
│ ├── Strong consistency (all queries)                             │
│ ├── Real-time sync for mobile/web (Native mode)                  │
│ ├── Multi-region replication built-in                            │
│ ├── Offline support (mobile/web SDKs cache locally)              │
│ └── Pay per operation + storage (no idle cost for zero traffic)   │
│                                                                       │
│ When to use Firestore:                                              │
│ ├── Mobile/web app backends (user profiles, chat, feeds)         │
│ ├── Session management, shopping carts                            │
│ ├── Content management (CMS data)                                 │
│ ├── IoT metadata (device state, configs)                          │
│ ├── Game state, leaderboards                                      │
│ └── Any workload with hierarchical, document-shaped data          │
│                                                                       │
│ When NOT to use Firestore:                                          │
│ ├── Relational data with complex JOINs → Cloud SQL or Spanner    │
│ ├── Analytics / warehousing → BigQuery                            │
│ ├── High-throughput time series → Bigtable                        │
│ ├── Simple key-value cache → Memorystore (Redis)                  │
│ └── ACID across arbitrary tables → Cloud SQL or Spanner           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

```
┌──────────────────────┬────────────────┬──────────────┬──────────────┐
│ Feature              │ GCP            │ AWS          │ Azure        │
├──────────────────────┼────────────────┼──────────────┼──────────────┤
│ Document DB          │ Firestore      │ DynamoDB     │ Cosmos DB    │
│ Serverless NoSQL     │ ✅ Yes          │ ✅ Yes        │ ✅ Yes        │
│ Real-time sync       │ ✅ Native mode  │ AppSync      │ Change Feed  │
│ Mobile SDK           │ Firebase SDK   │ Amplify      │ Cosmos SDK   │
│ Offline support      │ ✅ Built-in     │ Amplify DS   │ ❌ Limited    │
│ Security rules       │ ✅ Declarative  │ ❌ IAM-only   │ ❌ No         │
│ Multi-region         │ ✅ Automatic    │ Global Tables│ Multi-region │
│ Pricing model        │ Per operation  │ Per RCU/WCU  │ Per RU       │
│ Query flexibility    │ Limited        │ Limited      │ SQL-like     │
│ Max document size    │ 1 MiB          │ 400 KB       │ 2 MiB        │
└──────────────────────┴────────────────┴──────────────┴──────────────┘
```

### Pricing Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRESTORE PRICING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pay for what you use — no provisioned capacity:                    │
│                                                                       │
│ ┌─────────────────────────┬──────────────────────────────────────┐ │
│ │ Component               │ Price (approximate)                   │ │
│ ├─────────────────────────┼──────────────────────────────────────┤ │
│ │ Document reads           │ $0.06 per 100,000 reads              │ │
│ │ Document writes          │ $0.18 per 100,000 writes             │ │
│ │ Document deletes         │ $0.02 per 100,000 deletes            │ │
│ │ Stored data              │ $0.18 per GiB/month                  │ │
│ │ Network egress           │ Standard GCP rates                   │ │
│ └─────────────────────────┴──────────────────────────────────────┘ │
│                                                                       │
│ Free tier (per project/day):                                       │
│ ├── 50,000 reads                                                 │
│ ├── 20,000 writes                                                │
│ ├── 20,000 deletes                                               │
│ └── 1 GiB stored data                                            │
│                                                                       │
│ ⚡ Multi-region locations cost more than regional.                  │
│ ⚡ Reads include queries (each doc returned = 1 read).             │
│ ⚡ A query that returns 0 results = 1 read (minimum).             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Native Mode vs Datastore Mode

```
┌─────────────────────────────────────────────────────────────────────┐
│           NATIVE MODE vs DATASTORE MODE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Both are the SAME underlying Firestore engine.                     │
│ The "mode" controls which API and features are exposed.            │
│                                                                       │
│ ┌──────────────────────┬───────────────────┬──────────────────────┐│
│ │ Feature              │ Native Mode ✅     │ Datastore Mode       ││
│ ├──────────────────────┼───────────────────┼──────────────────────┤│
│ │ Real-time listeners  │ ✅ Yes              │ ❌ No                 ││
│ │ Offline support      │ ✅ Yes (mobile/web)│ ❌ No                 ││
│ │ Firebase SDK         │ ✅ Yes              │ ❌ No                 ││
│ │ Security rules       │ ✅ Yes              │ ❌ No (IAM only)      ││
│ │ Subcollections       │ ✅ Yes              │ ❌ No (ancestors)     ││
│ │ Collection group     │ ✅ Yes              │ ❌ No                 ││
│ │   queries            │                   │                      ││
│ │ Server-side client   │ ✅ Yes              │ ✅ Yes                ││
│ │   libraries          │                   │                      ││
│ │ Datastore API compat │ ❌ No               │ ✅ Yes                ││
│ │ Strong consistency   │ ✅ All queries      │ ✅ All queries        ││
│ │ Entity groups (legacy)│ ❌ Not applicable  │ ✅ Ancestor paths     ││
│ │ Terraform support    │ ✅ Yes              │ ✅ Yes                ││
│ │ Multiple databases   │ ✅ Yes              │ ✅ Yes                ││
│ │   per project        │                   │                      ││
│ └──────────────────────┴───────────────────┴──────────────────────┘│
│                                                                       │
│ Decision tree:                                                      │
│                                                                       │
│ Are you building a mobile/web app with real-time features?         │
│ ├── YES → Native mode ✅                                           │
│ └── NO                                                             │
│     ├── Migrating from legacy Datastore? → Datastore mode          │
│     ├── Server-to-server only (no client SDK)? → Either works      │
│     └── New project, no legacy? → Native mode ✅                   │
│                                                                       │
│ ⚠️ You choose the mode when creating the FIRST database in the    │
│    project. You CANNOT switch modes later (per database).          │
│    However, you can create additional named databases in either    │
│    mode within the same project (multi-database feature).          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Datastore Mode Legacy Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│           DATASTORE MODE TERMINOLOGY                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Datastore mode uses legacy terminology from the old Datastore      │
│ service. Here's the mapping:                                        │
│                                                                       │
│ ┌────────────────────┬───────────────────────────────┐             │
│ │ Datastore Term     │ Native Mode Equivalent         │             │
│ ├────────────────────┼───────────────────────────────┤             │
│ │ Kind               │ Collection                     │             │
│ │ Entity             │ Document                       │             │
│ │ Property           │ Field                          │             │
│ │ Key                │ Document ID                    │             │
│ │ Ancestor path      │ Subcollection hierarchy        │             │
│ │ Entity group       │ Document + subcollections      │             │
│ │ Namespace          │ Database (multi-database)      │             │
│ └────────────────────┴───────────────────────────────┘             │
│                                                                       │
│ If you have existing Datastore code:                                │
│ ├── Keep using Datastore mode — it's fully supported              │
│ ├── APIs remain stable (google.cloud.datastore)                   │
│ └── Consider migrating to Native mode for new features            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Data Model — Documents & Collections

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRESTORE DATA MODEL                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Everything in Firestore is either a DOCUMENT or a COLLECTION.      │
│                                                                       │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │                     DATABASE                                   │  │
│ │                                                               │  │
│ │   ┌─── Collection: "users" ──────────────────────────────┐   │  │
│ │   │                                                       │   │  │
│ │   │   ┌── Document: "user_abc" ───────────────────────┐  │   │  │
│ │   │   │ {                                              │  │   │  │
│ │   │   │   "name": "Alice",                            │  │   │  │
│ │   │   │   "email": "alice@example.com",               │  │   │  │
│ │   │   │   "age": 30,                                  │  │   │  │
│ │   │   │   "tags": ["admin", "developer"],             │  │   │  │
│ │   │   │   "address": {                                │  │   │  │
│ │   │   │     "city": "Mumbai",                         │  │   │  │
│ │   │   │     "country": "India"                        │  │   │  │
│ │   │   │   }                                           │  │   │  │
│ │   │   │ }                                              │  │   │  │
│ │   │   │                                                │  │   │  │
│ │   │   │   ┌── Subcollection: "orders" ─────────┐     │  │   │  │
│ │   │   │   │                                     │     │  │   │  │
│ │   │   │   │   ┌── Doc: "order_001" ──────┐    │     │  │   │  │
│ │   │   │   │   │ { "total": 150.00,       │    │     │  │   │  │
│ │   │   │   │   │   "status": "shipped" }  │    │     │  │   │  │
│ │   │   │   │   └──────────────────────────┘    │     │  │   │  │
│ │   │   │   │                                     │     │  │   │  │
│ │   │   │   │   ┌── Doc: "order_002" ──────┐    │     │  │   │  │
│ │   │   │   │   │ { "total": 89.99,        │    │     │  │   │  │
│ │   │   │   │   │   "status": "pending" }  │    │     │  │   │  │
│ │   │   │   │   └──────────────────────────┘    │     │  │   │  │
│ │   │   │   │                                     │     │  │   │  │
│ │   │   │   └────────────────────────────────────┘     │  │   │  │
│ │   │   └────────────────────────────────────────────────┘  │   │  │
│ │   │                                                       │   │  │
│ │   │   ┌── Document: "user_def" ───────────────────────┐  │   │  │
│ │   │   │ { "name": "Bob", "email": "bob@co.com" }      │  │   │  │
│ │   │   └────────────────────────────────────────────────┘  │   │  │
│ │   │                                                       │   │  │
│ │   └───────────────────────────────────────────────────────┘   │  │
│ │                                                               │  │
│ │   ┌─── Collection: "products" ───────────────────────────┐   │  │
│ │   │   ┌── Document: "prod_xyz" ──────────────────────┐   │   │  │
│ │   │   │ { "name": "Laptop", "price": 999.99 }        │   │   │  │
│ │   │   └──────────────────────────────────────────────┘   │   │  │
│ │   └───────────────────────────────────────────────────────┘   │  │
│ │                                                               │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Document Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│           DOCUMENT RULES & LIMITS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ A document:                                                         │
│ ├── Is identified by a unique ID (string) within its collection   │
│ ├── Contains fields (key-value pairs)                              │
│ ├── Can have subcollections (collections nested under it)          │
│ ├── Maximum size: 1 MiB (1,048,576 bytes)                         │
│ ├── Maximum depth: 100 levels (collection/document alternating)    │
│ ├── Document ID: 1-1500 bytes, UTF-8, no "/" or ".."              │
│ └── Can be schema-less — documents in the same collection         │
│     can have completely different fields                            │
│                                                                       │
│ Supported field types:                                              │
│ ├── String          "hello world"                                 │
│ ├── Number          42, 3.14 (64-bit IEEE 754)                    │
│ ├── Boolean         true, false                                    │
│ ├── Map             { "key": "value" } (nested object)            │
│ ├── Array           [1, "two", true] (mixed types OK)             │
│ ├── Null            null                                           │
│ ├── Timestamp       2024-01-15T10:30:00Z                          │
│ ├── GeoPoint        { latitude: 19.076, longitude: 72.877 }       │
│ ├── Reference       pointer to another document                    │
│ └── Bytes           binary data (up to 1 MiB per field)            │
│                                                                       │
│ Document path (address):                                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ /users/user_abc                         ← top-level doc       │  │
│ │ /users/user_abc/orders/order_001        ← subcollection doc   │  │
│ │ /users/user_abc/orders/order_001/items/item_a ← deeper nesting│  │
│ │                                                               │  │
│ │ Pattern: collection/document/collection/document/...         │  │
│ │ Always alternating: collection → document → collection → ... │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Documents do NOT contain other documents. They contain          │
│    subcollections which contain documents. The alternating          │
│    pattern is enforced.                                             │
│                                                                       │
│ ⚡ A document can exist without its parent document.                │
│    E.g., /users/user_abc/orders/order_001 can exist even if        │
│    /users/user_abc doesn't exist.                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Collections vs Subcollections

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHEN TO USE SUBCOLLECTIONS vs TOP-LEVEL                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Option 1: Subcollection (nested)                                   │
│ ┌──────────────────────────────────────────────────┐               │
│ │ /users/{userId}/orders/{orderId}                  │               │
│ │                                                   │               │
│ │ ✅ Natural parent-child relationship              │               │
│ │ ✅ Easy to get "all orders for user X"           │               │
│ │ ✅ Security rules can scope access per user      │               │
│ │ ❌ Harder to query "all orders across all users" │               │
│ │    (requires collection group query + index)     │               │
│ └──────────────────────────────────────────────────┘               │
│                                                                       │
│ Option 2: Top-level collection (flat)                              │
│ ┌──────────────────────────────────────────────────┐               │
│ │ /orders/{orderId}  ← with a "userId" field        │               │
│ │                                                   │               │
│ │ ✅ Easy to query "all orders" across everything  │               │
│ │ ✅ Simple flat structure                          │               │
│ │ ❌ Must manually filter by userId                │               │
│ │ ❌ Security rules need to check userId field     │               │
│ └──────────────────────────────────────────────────┘               │
│                                                                       │
│ Decision rule:                                                      │
│ ├── Data always accessed via parent? → Subcollection               │
│ ├── Data queried across all parents? → Top-level collection        │
│ └── Both? → Subcollection + collection group queries               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Creating a Firestore Database

### Console Walkthrough

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Firestore → Create Database                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Choose Mode                                                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ Native mode (recommended)                                   │   │
│ │   Real-time updates, offline support, Firebase SDK            │   │
│ │                                                               │   │
│ │ ○ Datastore mode                                              │   │
│ │   Server-side use only, Datastore API compatibility           │   │
│ │                                                               │   │
│ │ → Select "Native mode" for new projects ✅                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 2: Database ID                                                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Database ID: [(default)]                                      │   │
│ │                                                               │   │
│ │ • "(default)" = the default database (most projects use this)│   │
│ │ • Can also create named databases: "my-app-db", "staging"    │   │
│ │ • Named databases allow multiple DBs per project             │   │
│ │ • Named databases support different modes in same project    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 3: Location                                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Location type:                                                │   │
│ │ ○ Region: Single region (lower cost, lower latency)          │   │
│ │   Examples: us-east1, asia-south1, europe-west1              │   │
│ │                                                               │   │
│ │ ○ Multi-region: Replicated across multiple regions           │   │
│ │   Options:                                                    │   │
│ │   • nam5 (United States)                                     │   │
│ │   • eur3 (Europe)                                            │   │
│ │   • asia-southeast1 (Asia)                                   │   │
│ │                                                               │   │
│ │ ⚠️ Location CANNOT be changed after creation!                │   │
│ │ ⚠️ Must match App Engine region if App Engine is enabled     │   │
│ │ ⚠️ Multi-region is ~2x the cost of single region             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 4: Security Rules (Native mode only)                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ Start in production mode                                    │   │
│ │   → Deny all client reads/writes by default                  │   │
│ │   → You must write rules before clients can access           │   │
│ │                                                               │   │
│ │ ○ Start in test mode                                          │   │
│ │   → Allow all reads/writes for 30 days                       │   │
│ │   → Convenient for prototyping, but INSECURE                 │   │
│ │   → Expires automatically after 30 days                      │   │
│ │                                                               │   │
│ │ → For production: Always choose "production mode" ✅          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 5: Encryption (optional)                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ Google-managed encryption key (default) ✅                  │   │
│ │ ○ Customer-managed encryption key (CMEK)                      │   │
│ │   → Requires Cloud KMS key in same location                  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ [CREATE DATABASE]                                                    │
│                                                                       │
│ ⚡ Database is ready in ~30 seconds.                                │
│ ⚡ No provisioning, no waiting for instances.                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Multiple Databases per Project

```
┌─────────────────────────────────────────────────────────────────────┐
│           MULTI-DATABASE SUPPORT                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Since 2023, Firestore supports multiple databases per project.     │
│                                                                       │
│ Project: "my-saas-app"                                             │
│ ├── Database: "(default)" → Native mode, nam5                     │
│ │   └── Main application data                                     │
│ ├── Database: "analytics" → Datastore mode, us-east1              │
│ │   └── Server-side analytics pipeline                            │
│ └── Database: "staging"   → Native mode, us-central1              │
│     └── Staging environment data                                   │
│                                                                       │
│ Benefits:                                                           │
│ ├── Separate data for different environments or teams              │
│ ├── Different modes in same project                                │
│ ├── Independent security rules per database                       │
│ ├── Independent locations per database                             │
│ └── Separate billing metrics                                       │
│                                                                       │
│ Limits:                                                              │
│ ├── Max 100 databases per project                                  │
│ └── Each database is independent (no cross-database queries)       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Reading & Writing Data

### Writing Documents

```
┌─────────────────────────────────────────────────────────────────────┐
│           WRITE OPERATIONS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ There are four ways to write data:                                  │
│                                                                       │
│ 1. set() — Create or overwrite a document                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ // Node.js / Firebase SDK                                     │   │
│ │ const db = getFirestore();                                    │   │
│ │                                                               │   │
│ │ // Set with explicit ID                                       │   │
│ │ await setDoc(doc(db, "users", "user_abc"), {                  │   │
│ │   name: "Alice",                                              │   │
│ │   email: "alice@example.com",                                 │   │
│ │   age: 30,                                                    │   │
│ │   createdAt: serverTimestamp()                                │   │
│ │ });                                                           │   │
│ │                                                               │   │
│ │ // ⚠️ set() replaces the ENTIRE document                     │   │
│ │ // Use merge option to only update specified fields:          │   │
│ │ await setDoc(doc(db, "users", "user_abc"), {                  │   │
│ │   age: 31                                                     │   │
│ │ }, { merge: true });  // only updates age, keeps other fields│   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. add() — Create with auto-generated ID                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ const docRef = await addDoc(collection(db, "messages"), {     │   │
│ │   text: "Hello!",                                             │   │
│ │   sender: "user_abc",                                         │   │
│ │   timestamp: serverTimestamp()                                │   │
│ │ });                                                           │   │
│ │ console.log("New doc ID:", docRef.id);                        │   │
│ │ // ID looks like: "Xk2mF9v3pQr7sW1n"                        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. update() — Update specific fields (document must exist)         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ await updateDoc(doc(db, "users", "user_abc"), {               │   │
│ │   age: 31,                                                    │   │
│ │   "address.city": "Delhi",     // nested field update         │   │
│ │   tags: arrayUnion("manager"), // add to array                │   │
│ │   loginCount: increment(1)     // atomic increment            │   │
│ │ });                                                           │   │
│ │                                                               │   │
│ │ // ⚠️ Throws error if document doesn't exist                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. delete() — Delete a document                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ await deleteDoc(doc(db, "users", "user_abc"));                │   │
│ │                                                               │   │
│ │ // ⚠️ Deleting a document does NOT delete subcollections     │   │
│ │ // You must delete subcollection docs separately              │   │
│ │                                                               │   │
│ │ // Delete a specific field:                                   │   │
│ │ await updateDoc(doc(db, "users", "user_abc"), {               │   │
│ │   age: deleteField()                                          │   │
│ │ });                                                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Special field values (sentinels):                                   │
│ ├── serverTimestamp()  → server-set timestamp                     │
│ ├── increment(n)       → atomic counter increment/decrement       │
│ ├── arrayUnion(...)    → add elements to array (no duplicates)   │
│ ├── arrayRemove(...)   → remove elements from array              │
│ └── deleteField()      → remove a field from document             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Reading Documents

```
┌─────────────────────────────────────────────────────────────────────┐
│           READ OPERATIONS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Get a single document by path:                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ const docSnap = await getDoc(doc(db, "users", "user_abc"));   │   │
│ │                                                               │   │
│ │ if (docSnap.exists()) {                                       │   │
│ │   console.log("Data:", docSnap.data());                       │   │
│ │   // { name: "Alice", email: "alice@example.com", age: 30 } │   │
│ │ } else {                                                      │   │
│ │   console.log("No such document!");                           │   │
│ │ }                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. Get all documents in a collection:                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ const querySnap = await getDocs(collection(db, "users"));     │   │
│ │                                                               │   │
│ │ querySnap.forEach((doc) => {                                  │   │
│ │   console.log(doc.id, " => ", doc.data());                    │   │
│ │ });                                                           │   │
│ │                                                               │   │
│ │ // ⚠️ This reads EVERY document = costs 1 read per doc       │   │
│ │ // For large collections, use queries with filters            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. Get documents from a subcollection:                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ const orders = await getDocs(                                  │   │
│ │   collection(db, "users", "user_abc", "orders")               │   │
│ │ );                                                            │   │
│ │                                                               │   │
│ │ orders.forEach((doc) => {                                     │   │
│ │   console.log(doc.id, doc.data());                            │   │
│ │ });                                                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Server-side (Python / Admin SDK):                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ from google.cloud import firestore                            │   │
│ │                                                               │   │
│ │ db = firestore.Client()                                       │   │
│ │                                                               │   │
│ │ # Get single document                                         │   │
│ │ doc = db.collection("users").document("user_abc").get()        │   │
│ │ if doc.exists:                                                │   │
│ │     print(doc.to_dict())                                      │   │
│ │                                                               │   │
│ │ # Get all docs in collection                                  │   │
│ │ docs = db.collection("users").stream()                        │   │
│ │ for doc in docs:                                              │   │
│ │     print(f"{doc.id} => {doc.to_dict()}")                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Querying

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRESTORE QUERIES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Firestore queries are ALWAYS indexed. If no index exists for       │
│ your query, Firestore returns an error (not a slow scan).          │
│ This guarantees query performance proportional to the RESULT SET   │
│ size, not the total data size.                                      │
│                                                                       │
│ Query operators:                                                    │
│ ├── == (equal)                                                     │
│ ├── != (not equal)                                                 │
│ ├── <, <=, >, >= (range)                                           │
│ ├── in (match any in list, max 30 values)                          │
│ ├── not-in (exclude list, max 10 values)                           │
│ ├── array-contains (array has element)                              │
│ ├── array-contains-any (array has any of list, max 30)             │
│ └── orderBy, limit, startAt, startAfter, endAt, endBefore          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Simple Queries

```
┌─────────────────────────────────────────────────────────────────────┐
│           SIMPLE QUERIES (single field)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ // Equality                                                         │
│ const q1 = query(                                                   │
│   collection(db, "users"),                                          │
│   where("age", "==", 30)                                            │
│ );                                                                  │
│                                                                       │
│ // Range                                                            │
│ const q2 = query(                                                   │
│   collection(db, "users"),                                          │
│   where("age", ">=", 25),                                           │
│   where("age", "<=", 40)                                            │
│ );                                                                  │
│                                                                       │
│ // In (match any)                                                   │
│ const q3 = query(                                                   │
│   collection(db, "products"),                                       │
│   where("category", "in", ["electronics", "books", "toys"])         │
│ );                                                                  │
│                                                                       │
│ // Array contains                                                   │
│ const q4 = query(                                                   │
│   collection(db, "users"),                                          │
│   where("tags", "array-contains", "admin")                          │
│ );                                                                  │
│                                                                       │
│ // Order and limit                                                  │
│ const q5 = query(                                                   │
│   collection(db, "users"),                                          │
│   orderBy("createdAt", "desc"),                                     │
│   limit(10)                                                         │
│ );                                                                  │
│                                                                       │
│ // Execute any query:                                               │
│ const snapshot = await getDocs(q1);                                 │
│ snapshot.forEach(doc => console.log(doc.id, doc.data()));           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Compound Queries

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMPOUND QUERIES (multiple fields)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ // Multiple equality filters — automatic index works               │
│ const q = query(                                                    │
│   collection(db, "orders"),                                         │
│   where("status", "==", "shipped"),                                 │
│   where("region", "==", "asia")                                     │
│ );                                                                  │
│                                                                       │
│ // Equality + Range — needs composite index                        │
│ const q2 = query(                                                   │
│   collection(db, "orders"),                                         │
│   where("status", "==", "shipped"),                                 │
│   where("total", ">", 100),                                        │
│   orderBy("total")                                                  │
│ );                                                                  │
│                                                                       │
│ ⚠️ QUERY LIMITATIONS:                                               │
│                                                                       │
│ 1. Range filters on ONE field only:                                │
│    ❌ where("age", ">", 25), where("salary", "<", 100000)          │
│    ✅ where("status", "==", "active"), where("age", ">", 25)       │
│    ✅ (since Firestore 2024: range on multiple fields is now        │
│        supported with composite indexes, but with limitations)     │
│                                                                       │
│ 2. orderBy must match the range field:                             │
│    ❌ where("age", ">", 25), orderBy("name")                       │
│    ✅ where("age", ">", 25), orderBy("age"), orderBy("name")       │
│                                                                       │
│ 3. Cannot combine not-in/!= with in/array-contains-any:           │
│    ❌ where("status", "!=", "draft"), where("region", "in", [...]) │
│                                                                       │
│ 4. OR queries (since 2023):                                        │
│    const q = query(                                                 │
│      collection(db, "orders"),                                      │
│      or(                                                            │
│        where("status", "==", "pending"),                            │
│        where("status", "==", "processing")                          │
│      )                                                              │
│    );                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Collection Group Queries

```
┌─────────────────────────────────────────────────────────────────────┐
│           COLLECTION GROUP QUERIES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: You have subcollections under different parents:           │
│                                                                       │
│ /users/alice/orders/order001                                        │
│ /users/bob/orders/order002                                          │
│ /users/carol/orders/order003                                        │
│                                                                       │
│ How to query ALL orders across ALL users?                           │
│                                                                       │
│ Solution: Collection group query                                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ // Query ALL "orders" subcollections across all parents       │   │
│ │ const q = query(                                              │   │
│ │   collectionGroup(db, "orders"),                              │   │
│ │   where("status", "==", "pending"),                           │   │
│ │   orderBy("createdAt", "desc"),                               │   │
│ │   limit(50)                                                   │   │
│ │ );                                                            │   │
│ │                                                               │   │
│ │ const snapshot = await getDocs(q);                            │   │
│ │ snapshot.forEach(doc => {                                     │   │
│ │   console.log(doc.ref.path); // includes full parent path    │   │
│ │   // e.g., "users/alice/orders/order001"                     │   │
│ │ });                                                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ Collection group queries require a composite index.              │
│    Firestore console shows the exact index to create when you      │
│    run a query that needs one.                                      │
│                                                                       │
│ ⚠️ Security rules: Must explicitly allow collection group queries. │
│    match /{path=**}/orders/{orderId} { ... }                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Pagination (Cursors)

```
┌─────────────────────────────────────────────────────────────────────┐
│           PAGINATION WITH CURSORS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Firestore uses cursor-based pagination (no offset/skip):           │
│                                                                       │
│ // Page 1                                                           │
│ const first = query(                                                │
│   collection(db, "products"),                                       │
│   orderBy("name"),                                                  │
│   limit(25)                                                         │
│ );                                                                  │
│ const firstPage = await getDocs(first);                             │
│ const lastDoc = firstPage.docs[firstPage.docs.length - 1];         │
│                                                                       │
│ // Page 2 — start after last document of page 1                    │
│ const next = query(                                                 │
│   collection(db, "products"),                                       │
│   orderBy("name"),                                                  │
│   startAfter(lastDoc),                                              │
│   limit(25)                                                         │
│ );                                                                  │
│ const secondPage = await getDocs(next);                             │
│                                                                       │
│ ⚡ No "total count" — you can't know total docs without            │
│    reading them all (which costs reads). Maintain a counter        │
│    document if you need totals.                                     │
│                                                                       │
│ ⚡ No offset-based pagination — can't jump to "page 5".            │
│    Always navigate forward/backward from a cursor.                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Indexes

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRESTORE INDEXES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ EVERY Firestore query must be backed by an index.                  │
│ If no index exists, the query FAILS (not slow, FAILS).             │
│ This guarantees O(result_set) performance.                         │
│                                                                       │
│ Two types of indexes:                                               │
│                                                                       │
│ 1. SINGLE-FIELD INDEXES (automatic ✅)                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Firestore automatically creates indexes for:                  │   │
│ │ • Every field (ascending)                                     │   │
│ │ • Every field (descending)                                    │   │
│ │ • Every array field (array-contains)                          │   │
│ │                                                               │   │
│ │ These power simple queries:                                   │   │
│ │ • where("status", "==", "active")                            │   │
│ │ • where("age", ">=", 25)                                     │   │
│ │ • where("tags", "array-contains", "admin")                   │   │
│ │ • orderBy("createdAt", "desc")                               │   │
│ │                                                               │   │
│ │ You never need to create these — they're built-in.           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. COMPOSITE INDEXES (manual — you create these)                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Needed for queries that filter/sort on MULTIPLE fields.       │   │
│ │                                                               │   │
│ │ Example: where("status", "==", "active")                     │   │
│ │          + orderBy("createdAt", "desc")                      │   │
│ │                                                               │   │
│ │ This needs a composite index on (status ASC, createdAt DESC).│   │
│ │                                                               │   │
│ │ How to create:                                                │   │
│ │ Option A: Run the query → get error → click link in error    │   │
│ │           message → auto-creates the index ✅ (easiest)       │   │
│ │                                                               │   │
│ │ Option B: Console → Firestore → Indexes → Add Index          │   │
│ │           • Collection: "orders"                              │   │
│ │           • Fields: status (Ascending), createdAt (Descending)│   │
│ │           • Query scope: Collection or Collection group       │   │
│ │                                                               │   │
│ │ Option C: firestore.indexes.json (deployed via Firebase CLI)  │   │
│ │   {                                                           │   │
│ │     "indexes": [{                                             │   │
│ │       "collectionGroup": "orders",                            │   │
│ │       "queryScope": "COLLECTION",                             │   │
│ │       "fields": [                                             │   │
│ │         { "fieldPath": "status", "order": "ASCENDING" },     │   │
│ │         { "fieldPath": "createdAt", "order": "DESCENDING" }  │   │
│ │       ]                                                       │   │
│ │     }]                                                        │   │
│ │   }                                                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Index limits per database:                                          │
│ ├── 200 composite indexes                                         │
│ ├── 500 single-field index exemptions                             │
│ └── Building time: seconds to minutes depending on data size      │
│                                                                       │
│ Index exemptions:                                                   │
│ ├── Disable auto-indexing for specific fields                      │
│ ├── Useful for: large map fields, fields never queried            │
│ ├── Saves storage and write costs                                  │
│ └── Console → Indexes → Single-field → Add exemption              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Index Costs

```
┌─────────────────────────────────────────────────────────────────────┐
│           INDEX STORAGE COSTS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Indexes consume storage and add write cost:                        │
│                                                                       │
│ When you write a document with 10 fields:                          │
│ ├── Document storage: 1 write                                     │
│ ├── Single-field indexes: 10 ascending + 10 descending = 20       │
│ │   index entries (automatic, can't avoid)                         │
│ └── Composite indexes: 1 entry per composite index that covers     │
│     this document                                                   │
│                                                                       │
│ Total index entries per document write can be significant!         │
│                                                                       │
│ Limit: 40,000 index entries per document                           │
│ (combining single-field + composite)                                │
│                                                                       │
│ Optimization:                                                       │
│ ├── Exempt fields you never query on (large maps, blobs)          │
│ ├── Don't store unnecessary fields in documents                    │
│ └── Delete unused composite indexes                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Real-Time Listeners

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-TIME LISTENERS (Native Mode Only)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The killer feature of Firestore Native mode: your app gets         │
│ notified instantly when data changes. No polling needed.            │
│                                                                       │
│ Listen to a single document:                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ const unsub = onSnapshot(doc(db, "users", "user_abc"),        │   │
│ │   (docSnap) => {                                              │   │
│ │     if (docSnap.exists()) {                                   │   │
│ │       console.log("Current data:", docSnap.data());           │   │
│ │       // Called immediately with current data                 │   │
│ │       // Called again whenever the document changes           │   │
│ │     }                                                         │   │
│ │   },                                                          │   │
│ │   (error) => {                                                │   │
│ │     console.error("Listen failed:", error);                   │   │
│ │   }                                                           │   │
│ │ );                                                            │   │
│ │                                                               │   │
│ │ // Stop listening when done:                                  │   │
│ │ unsub();                                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Listen to a query (collection):                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ const q = query(                                              │   │
│ │   collection(db, "messages"),                                 │   │
│ │   where("room", "==", "general"),                             │   │
│ │   orderBy("timestamp", "desc"),                               │   │
│ │   limit(50)                                                   │   │
│ │ );                                                            │   │
│ │                                                               │   │
│ │ const unsub = onSnapshot(q, (snapshot) => {                   │   │
│ │   // Full snapshot with all matching docs                     │   │
│ │   snapshot.forEach(doc => console.log(doc.data()));           │   │
│ │                                                               │   │
│ │   // Incremental changes since last snapshot                  │   │
│ │   snapshot.docChanges().forEach(change => {                   │   │
│ │     if (change.type === "added") { /* new doc */ }            │   │
│ │     if (change.type === "modified") { /* changed doc */ }     │   │
│ │     if (change.type === "removed") { /* removed doc */ }      │   │
│ │   });                                                         │   │
│ │                                                               │   │
│ │   // Check if data came from server or local cache            │   │
│ │   const source = snapshot.metadata.fromCache                  │   │
│ │     ? "local cache" : "server";                               │   │
│ │ });                                                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ How it works under the hood:                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Client A                    Firestore                        │   │
│ │  ┌───────┐                  ┌──────────┐                     │   │
│ │  │ App   │──── listen ────→│          │                     │   │
│ │  │       │←── snapshot ────│          │                     │   │
│ │  │       │                  │          │                     │   │
│ │  │       │                  │          │                     │   │
│ │  │       │                  │          │   Client B writes   │   │
│ │  │       │                  │          │←── write ──┐       │   │
│ │  │       │←── snapshot ────│          │            │       │   │
│ │  │       │   (change!)      │          │       ┌───────┐   │   │
│ │  └───────┘                  └──────────┘       │ App B │   │   │
│ │                                                 └───────┘   │   │
│ │  ⚡ Typical latency: < 100ms for same-region              │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Cost: Each listener creates one persistent connection.             │
│ Reads are billed only for documents received (initial +            │
│ each change). Maintaining the connection itself is free.           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Security Rules (Native Mode)

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRESTORE SECURITY RULES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Security rules control client-side access (Firebase SDK).           │
│ They do NOT apply to server-side Admin SDK access.                  │
│                                                                       │
│ Rules are written in a custom declarative language:                 │
│                                                                       │
│ rules_version = '2';                                                │
│ service cloud.firestore {                                           │
│   match /databases/{database}/documents {                           │
│                                                                       │
│     // Rule 1: Users can read/write their own profile              │
│     match /users/{userId} {                                         │
│       allow read, write: if request.auth != null                    │
│                           && request.auth.uid == userId;            │
│     }                                                               │
│                                                                       │
│     // Rule 2: Anyone authenticated can read products              │
│     match /products/{productId} {                                   │
│       allow read: if request.auth != null;                          │
│       allow write: if request.auth.token.admin == true;             │
│     }                                                               │
│                                                                       │
│     // Rule 3: Users can read/write their own orders               │
│     match /users/{userId}/orders/{orderId} {                        │
│       allow read, write: if request.auth.uid == userId;             │
│     }                                                               │
│                                                                       │
│     // Rule 4: Collection group query access                       │
│     match /{path=**}/orders/{orderId} {                             │
│       allow read: if request.auth.token.admin == true;              │
│     }                                                               │
│   }                                                                 │
│ }                                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Rule Building Blocks

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY RULES REFERENCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Available variables:                                                │
│ ┌────────────────────────────┬──────────────────────────────────┐  │
│ │ Variable                   │ Description                       │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ request.auth               │ Auth context (null if unauthed)   │  │
│ │ request.auth.uid           │ Firebase Auth user ID             │  │
│ │ request.auth.token         │ JWT claims (email, custom claims) │  │
│ │ request.resource.data      │ Incoming document data (writes)   │  │
│ │ resource.data              │ Existing document data (before)   │  │
│ │ request.time               │ Request timestamp                 │  │
│ │ request.method             │ get, list, create, update, delete │  │
│ └────────────────────────────┴──────────────────────────────────┘  │
│                                                                       │
│ Operations (granular permissions):                                  │
│ ├── read  = get + list                                            │
│ ├── write = create + update + delete                              │
│ ├── get   = read a single document                                │
│ ├── list  = query a collection                                     │
│ ├── create = write a new document                                  │
│ ├── update = modify existing document                              │
│ └── delete = remove a document                                     │
│                                                                       │
│ Common patterns:                                                    │
│                                                                       │
│ // Validate data on write                                          │
│ allow create: if request.resource.data.name is string              │
│               && request.resource.data.name.size() <= 100          │
│               && request.resource.data.keys().hasAll(              │
│                    ['name', 'email']);                               │
│                                                                       │
│ // Prevent field modification                                      │
│ allow update: if request.resource.data.createdBy                   │
│               == resource.data.createdBy;                           │
│  // createdBy can't be changed after creation                      │
│                                                                       │
│ // Rate limiting (check timestamp)                                 │
│ allow create: if request.time >                                    │
│   resource.data.lastWrite + duration.value(1, 's');                │
│                                                                       │
│ // Check another document (function)                               │
│ function isAdmin() {                                               │
│   return get(/databases/$(database)/documents/                     │
│     admins/$(request.auth.uid)).data.active == true;               │
│ }                                                                  │
│ allow write: if isAdmin();                                         │
│                                                                       │
│ ⚠️ get() in rules counts as a READ (billed).                      │
│ ⚠️ Max 10 get() calls per rule evaluation.                        │
│ ⚠️ Rules are NOT filters — they allow/deny, not narrow results.   │
│                                                                       │
│ Deploy rules:                                                       │
│ ├── Console → Firestore → Rules → Edit & Publish                  │
│ ├── Firebase CLI: firebase deploy --only firestore:rules           │
│ └── File: firestore.rules (in your project root)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Transactions & Batched Writes

```
┌─────────────────────────────────────────────────────────────────────┐
│           TRANSACTIONS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Transactions let you read-then-write atomically.                   │
│ If another client modifies the data mid-transaction, Firestore     │
│ retries your transaction (up to 5 times by default).               │
│                                                                       │
│ Use case: Transfer money, decrement inventory, ensure consistency  │
│                                                                       │
│ // Atomic transfer between two accounts                            │
│ await runTransaction(db, async (transaction) => {                   │
│   // Step 1: Read both accounts                                    │
│   const fromSnap = await transaction.get(                           │
│     doc(db, "accounts", "alice")                                    │
│   );                                                                │
│   const toSnap = await transaction.get(                             │
│     doc(db, "accounts", "bob")                                      │
│   );                                                                │
│                                                                       │
│   const fromBalance = fromSnap.data().balance;                      │
│   const toBalance = toSnap.data().balance;                          │
│                                                                       │
│   if (fromBalance < 100) {                                          │
│     throw new Error("Insufficient funds");                          │
│   }                                                                 │
│                                                                       │
│   // Step 2: Write both updates                                    │
│   transaction.update(doc(db, "accounts", "alice"), {                │
│     balance: fromBalance - 100                                      │
│   });                                                               │
│   transaction.update(doc(db, "accounts", "bob"), {                  │
│     balance: toBalance + 100                                        │
│   });                                                               │
│ });                                                                 │
│                                                                       │
│ Transaction rules:                                                  │
│ ├── All reads MUST come before all writes                          │
│ ├── Max 500 documents per transaction                              │
│ ├── Transaction function must be idempotent (retries happen)       │
│ ├── Max execution time: 270 seconds (server SDK)                   │
│ ├── Transactions lock documents optimistically (fail on conflict)  │
│ └── Do NOT perform side effects inside transaction functions       │
│     (e.g., sending emails) — they may run multiple times           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Batched Writes

```
┌─────────────────────────────────────────────────────────────────────┐
│           BATCHED WRITES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Batched writes let you perform multiple writes atomically           │
│ WITHOUT reading first (unlike transactions).                       │
│                                                                       │
│ const batch = writeBatch(db);                                       │
│                                                                       │
│ // Set (create or overwrite)                                        │
│ batch.set(doc(db, "users", "user_new"), {                           │
│   name: "Charlie",                                                  │
│   email: "charlie@example.com"                                      │
│ });                                                                 │
│                                                                       │
│ // Update existing                                                  │
│ batch.update(doc(db, "users", "user_abc"), {                        │
│   lastLogin: serverTimestamp()                                      │
│ });                                                                 │
│                                                                       │
│ // Delete                                                           │
│ batch.delete(doc(db, "users", "user_old"));                         │
│                                                                       │
│ // Commit atomically — all succeed or all fail                     │
│ await batch.commit();                                               │
│                                                                       │
│ ├── Max 500 operations per batch                                  │
│ ├── No reads — use transactions if you need read-then-write       │
│ ├── Atomic — all operations succeed or none do                    │
│ └── Faster than individual writes (single RPC call)                │
│                                                                       │
│ Transaction vs Batch:                                               │
│ ┌──────────────────────┬──────────────────┬──────────────────────┐ │
│ │ Feature              │ Transaction      │ Batched Write        │ │
│ ├──────────────────────┼──────────────────┼──────────────────────┤ │
│ │ Reads                │ ✅ Yes            │ ❌ No                 │ │
│ │ Writes               │ ✅ Yes            │ ✅ Yes                │ │
│ │ Atomic               │ ✅ Yes            │ ✅ Yes                │ │
│ │ Retries on conflict  │ ✅ Automatic      │ N/A (no reads)       │ │
│ │ Max operations       │ 500              │ 500                  │ │
│ │ Use case             │ Read-then-write  │ Write-only bulk ops  │ │
│ └──────────────────────┴──────────────────┴──────────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Offline Support & Multi-Tab

```
┌─────────────────────────────────────────────────────────────────────┐
│           OFFLINE SUPPORT (Native Mode Only)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Firestore automatically caches data on the client device.           │
│ When the device goes offline, reads come from cache and             │
│ writes are queued locally.                                          │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  ┌─────────────┐      ONLINE      ┌──────────────┐          │   │
│ │  │   Client    │ ←──────────────→ │  Firestore   │          │   │
│ │  │  ┌───────┐  │                  │  (Server)    │          │   │
│ │  │  │ Cache │  │                  └──────────────┘          │   │
│ │  │  └───────┘  │                                             │   │
│ │  └─────────────┘                                             │   │
│ │                                                               │   │
│ │  ┌─────────────┐      OFFLINE     ┌──────────────┐          │   │
│ │  │   Client    │ ──── ✗ ────── → │  Firestore   │          │   │
│ │  │  ┌───────┐  │                  │  (Server)    │          │   │
│ │  │  │ Cache │  │ ← reads         └──────────────┘          │   │
│ │  │  │       │  │ ← writes queued                            │   │
│ │  │  └───────┘  │                                             │   │
│ │  └─────────────┘                                             │   │
│ │                                                               │   │
│ │  ┌─────────────┐      BACK ONLINE ┌──────────────┐          │   │
│ │  │   Client    │ ←──────────────→ │  Firestore   │          │   │
│ │  │  ┌───────┐  │ queued writes   │  (Server)    │          │   │
│ │  │  │ Cache │  │ ───────────────→│  synced!     │          │   │
│ │  │  └───────┘  │                  └──────────────┘          │   │
│ │  └─────────────┘                                             │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Platforms and cache:                                                │
│ ├── Web: IndexedDB cache (configurable size, default 40 MB)        │
│ ├── iOS: SQLite on-device cache                                    │
│ ├── Android: SQLite on-device cache                                │
│ └── Flutter: Dart-level in-memory + persistent cache               │
│                                                                       │
│ Enable offline persistence (web — enabled by default on mobile):   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ // Web — enable persistence                                   │   │
│ │ import { enableIndexedDbPersistence } from "firebase/firestore";  │
│ │ enableIndexedDbPersistence(db).catch((err) => {               │   │
│ │   if (err.code === 'failed-precondition') {                   │   │
│ │     // Multiple tabs open — only one can have persistence    │   │
│ │   } else if (err.code === 'unimplemented') {                 │   │
│ │     // Browser doesn't support IndexedDB                     │   │
│ │   }                                                           │   │
│ │ });                                                           │   │
│ │                                                               │   │
│ │ // Multi-tab persistence (since v9.8+):                      │   │
│ │ import { enableMultiTabIndexedDbPersistence } from             │   │
│ │   "firebase/firestore";                                       │   │
│ │ enableMultiTabIndexedDbPersistence(db);                       │   │
│ │ // Allows persistence across multiple browser tabs            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ Offline writes are queued but NOT guaranteed.                   │
│    If the app is killed before reconnecting, queued writes         │
│    are persisted to disk and retried on next app launch.           │
│                                                                       │
│ ⚠️ Cache-first behavior: snapshot listeners will first deliver     │
│    cached results, then update when server data arrives.           │
│    Check snapshot.metadata.fromCache to distinguish.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Backups & Point-in-Time Recovery

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKUPS & PITR                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. SCHEDULED BACKUPS                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console → Firestore → Backups → Create backup schedule        │   │
│ │                                                               │   │
│ │ • Frequency: Daily or Weekly                                  │   │
│ │ • Retention: 1 day to 14 weeks                               │   │
│ │ • Location: Same as database                                  │   │
│ │                                                               │   │
│ │ Backups are full database snapshots.                           │   │
│ │ Restore creates a NEW database (cannot overwrite existing).  │   │
│ │                                                               │   │
│ │ gcloud CLI:                                                   │   │
│ │ gcloud firestore backups schedules create \                   │   │
│ │   --database="(default)" \                                    │   │
│ │   --recurrence=daily \                                        │   │
│ │   --retention=7d                                              │   │
│ │                                                               │   │
│ │ Restore from backup:                                          │   │
│ │ gcloud firestore databases restore \                          │   │
│ │   --source-backup=projects/my-proj/locations/nam5/\          │   │
│ │     backups/20240115-daily \                                  │   │
│ │   --destination-database=restored-db                          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. POINT-IN-TIME RECOVERY (PITR)                                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Restore your database to any point in time within the        │   │
│ │ retention window (up to 7 days).                              │   │
│ │                                                               │   │
│ │ Must be enabled when creating the database:                   │   │
│ │ gcloud firestore databases create \                           │   │
│ │   --location=nam5 \                                           │   │
│ │   --type=FIRESTORE_NATIVE \                                   │   │
│ │   --enable-pitr                                               │   │
│ │                                                               │   │
│ │ Restore to specific time:                                     │   │
│ │ gcloud firestore databases restore \                          │   │
│ │   --source-database="(default)" \                             │   │
│ │   --snapshot-time="2024-01-15T10:30:00Z" \                   │   │
│ │   --destination-database=recovered-db                         │   │
│ │                                                               │   │
│ │ ⚠️ PITR increases storage costs (~2x for 7-day window).      │   │
│ │ ⚠️ Restore always creates a NEW database.                    │   │
│ │ ⚠️ Cannot enable PITR on an existing database — must         │   │
│ │    create a new database with PITR enabled.                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. EXPORT/IMPORT (legacy — still useful)                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Export to Cloud Storage (for BigQuery analysis, migration):   │   │
│ │                                                               │   │
│ │ gcloud firestore export gs://my-bucket/firestore-export \     │   │
│ │   --collection-ids=users,orders                               │   │
│ │                                                               │   │
│ │ Import from Cloud Storage:                                    │   │
│ │ gcloud firestore import gs://my-bucket/firestore-export       │   │
│ │                                                               │   │
│ │ ⚠️ Export is NOT a point-in-time snapshot if data is          │   │
│ │    being written during export. Use backups/PITR for          │   │
│ │    consistent snapshots.                                      │   │
│ │ ⚡ Exports can be loaded into BigQuery for analytics.         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Backup vs PITR vs Export:                                           │
│ ┌──────────────────┬───────────────┬──────────────┬──────────────┐│
│ │ Feature          │ Backup        │ PITR         │ Export       ││
│ ├──────────────────┼───────────────┼──────────────┼──────────────┤│
│ │ Consistency      │ ✅ Consistent  │ ✅ Consistent │ ⚠️ Eventual  ││
│ │ Granularity      │ Daily/Weekly  │ Any second   │ On-demand    ││
│ │ Restore target   │ New database  │ New database │ Same/new DB  ││
│ │ Retention        │ Up to 14 weeks│ Up to 7 days │ In GCS bucket││
│ │ Cost             │ Storage only  │ ~2x storage  │ GCS + ops    ││
│ │ BigQuery compat  │ ❌ No          │ ❌ No         │ ✅ Yes        ││
│ └──────────────────┴───────────────┴──────────────┴──────────────┘│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Performance & Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERFORMANCE BEST PRACTICES                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. DOCUMENT DESIGN                                                 │
│ ├── Keep documents small (< 10 KB ideally, max 1 MiB)             │
│ ├── Denormalize data — duplicate for read performance              │
│ │   Instead of joining users + orders, embed user name in order   │
│ ├── Avoid deeply nested subcollections (hard to query/delete)      │
│ ├── Use maps for small related data, subcollections for large     │
│ └── Don't store large binary data — use Cloud Storage + reference │
│                                                                       │
│ 2. QUERY OPTIMIZATION                                              │
│ ├── Always use indexes (Firestore enforces this anyway)            │
│ ├── Use limit() to cap result size                                 │
│ ├── Use cursors for pagination instead of offset                   │
│ ├── Avoid "fan-out" queries (reading too many documents)           │
│ └── Use collection group queries sparingly (cost = all matching    │
│     docs across all subcollections)                                 │
│                                                                       │
│ 3. WRITE OPTIMIZATION                                              │
│ ├── Max write rate per document: 1 write/second (sustained)        │
│ │   → For counters, use distributed counters (shard across N     │
│ │     sub-documents, sum on read)                                 │
│ ├── Batch writes for bulk operations (max 500 per batch)           │
│ ├── Use server timestamps instead of client timestamps             │
│ ├── Don't write to sequential document IDs (hotspot risk)         │
│ └── Ramp up traffic gradually for new collections                  │
│     (Firestore auto-scales, but needs warm-up time)               │
│                                                                       │
│ 4. DISTRIBUTED COUNTERS (solving the 1 write/sec limit)           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ Problem: Like counter on a popular post                       │   │
│ │ ❌ Direct: 1 doc = 1 write/sec max = bottleneck              │   │
│ │                                                               │   │
│ │ Solution: Shard the counter across N documents                │   │
│ │                                                               │   │
│ │ /counters/post_likes/shards/0  → { count: 42 }              │   │
│ │ /counters/post_likes/shards/1  → { count: 38 }              │   │
│ │ /counters/post_likes/shards/2  → { count: 55 }              │   │
│ │ ...                                                           │   │
│ │ /counters/post_likes/shards/9  → { count: 31 }              │   │
│ │                                                               │   │
│ │ Write: Pick random shard, increment(1) → 10 writes/sec      │   │
│ │ Read: Sum all shards → total = 42+38+55+...+31               │   │
│ │                                                               │   │
│ │ 10 shards = 10 writes/sec, 100 shards = 100 writes/sec     │   │
│ │ Trade-off: Read cost increases (N reads to get total)         │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 5. HOTSPOT AVOIDANCE                                               │
│ ├── Don't use monotonically increasing IDs (timestamps, 1,2,3)    │
│ ├── Let Firestore auto-generate IDs (well-distributed)             │
│ ├── Avoid writing to a single collection at very high rate         │
│ │   without gradual ramp-up                                        │
│ └── The "500/50/5" rule: start at 500 ops/sec per collection,     │
│     Firestore auto-scales by 50% every 5 minutes                  │
│                                                                       │
│ 6. COST OPTIMIZATION                                               │
│ ├── Use get() instead of onSnapshot() if you don't need          │
│ │   real-time (listeners keep connections open)                    │
│ ├── Use select() to return only specific fields (still 1 read     │
│ │   per doc but less bandwidth)                                    │
│ ├── Exempt unused fields from indexing                             │
│ ├── Set appropriate TTL or lifecycle to delete old documents       │
│ ├── Use regional over multi-region if HA isn't critical            │
│ └── Monitor with Cloud Monitoring → Firestore usage metrics       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### TTL (Time-to-Live) Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           TTL POLICIES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automatically delete documents after a specified time.              │
│ Great for session data, logs, temporary tokens.                    │
│                                                                       │
│ Setup:                                                              │
│ 1. Add a timestamp field to your documents (e.g., "expiresAt")     │
│ 2. Create a TTL policy on that field                               │
│                                                                       │
│ Console → Firestore → TTL → Create Policy                          │
│ • Collection group: "sessions"                                     │
│ • Timestamp field: "expiresAt"                                     │
│                                                                       │
│ gcloud CLI:                                                         │
│ gcloud firestore fields ttls update expiresAt \                     │
│   --collection-group=sessions \                                     │
│   --enable-ttl                                                      │
│                                                                       │
│ Document example:                                                   │
│ {                                                                   │
│   "userId": "abc",                                                  │
│   "token": "xyz123",                                                │
│   "expiresAt": Timestamp(2024-01-15T10:30:00Z)                     │
│ }                                                                   │
│ // Auto-deleted shortly after expiresAt passes                     │
│                                                                       │
│ ⚠️ Deletion is NOT immediate — typically within 24 hours          │
│    after the TTL timestamp passes.                                  │
│ ⚠️ TTL deletes are free (no delete operation cost).                │
│ ⚠️ Documents without the TTL field are not affected.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Terraform & CLI

### Terraform

```hcl
# ─────────────────────────────────────────────────────────────
# Firestore Database (Native Mode)
# ─────────────────────────────────────────────────────────────

resource "google_firestore_database" "main" {
  project     = "my-gcp-project"
  name        = "(default)"
  location_id = "nam5"  # Multi-region US
  type        = "FIRESTORE_NATIVE"

  # Optional: Enable PITR
  point_in_time_recovery_enablement = "POINT_IN_TIME_RECOVERY_ENABLED"

  # Optional: CMEK
  # cmek_config {
  #   kms_key_name = "projects/my-proj/locations/nam5/keyRings/kr/cryptoKeys/key"
  # }

  # Optional: Delete protection
  delete_protection_state = "DELETE_PROTECTION_ENABLED"
}

# ─────────────────────────────────────────────────────────────
# Named Database (Datastore Mode)
# ─────────────────────────────────────────────────────────────

resource "google_firestore_database" "analytics" {
  project     = "my-gcp-project"
  name        = "analytics"
  location_id = "us-east1"  # Regional
  type        = "DATASTORE_MODE"
}

# ─────────────────────────────────────────────────────────────
# Composite Index
# ─────────────────────────────────────────────────────────────

resource "google_firestore_index" "orders_status_date" {
  project    = "my-gcp-project"
  database   = google_firestore_database.main.name
  collection = "orders"

  fields {
    field_path = "status"
    order      = "ASCENDING"
  }

  fields {
    field_path = "createdAt"
    order      = "DESCENDING"
  }

  query_scope = "COLLECTION"
}

# ─────────────────────────────────────────────────────────────
# Collection Group Index (for subcollection queries)
# ─────────────────────────────────────────────────────────────

resource "google_firestore_index" "all_orders_by_status" {
  project    = "my-gcp-project"
  database   = google_firestore_database.main.name
  collection = "orders"

  fields {
    field_path = "status"
    order      = "ASCENDING"
  }

  fields {
    field_path = "total"
    order      = "DESCENDING"
  }

  query_scope = "COLLECTION_GROUP"
}

# ─────────────────────────────────────────────────────────────
# Single-Field Index Exemption
# ─────────────────────────────────────────────────────────────

resource "google_firestore_field" "metadata_exempt" {
  project    = "my-gcp-project"
  database   = google_firestore_database.main.name
  collection = "events"
  field      = "metadata"

  index_config {}  # Empty = exempt from all auto-indexing
}

# ─────────────────────────────────────────────────────────────
# TTL Policy
# ─────────────────────────────────────────────────────────────

resource "google_firestore_field" "sessions_ttl" {
  project    = "my-gcp-project"
  database   = google_firestore_database.main.name
  collection = "sessions"
  field      = "expiresAt"

  ttl_config {}  # Enable TTL on this field
}

# ─────────────────────────────────────────────────────────────
# Backup Schedule
# ─────────────────────────────────────────────────────────────

resource "google_firestore_backup_schedule" "daily" {
  project  = "my-gcp-project"
  database = google_firestore_database.main.name

  retention = "604800s"  # 7 days in seconds

  daily_recurrence {}  # or weekly_recurrence { day = "MONDAY" }
}
```

### gcloud CLI Reference

```bash
# ─────────────────────────────────────────────────────────────
# Database Management
# ─────────────────────────────────────────────────────────────

# Create database (Native mode, multi-region US)
gcloud firestore databases create \
  --location=nam5 \
  --type=FIRESTORE_NATIVE \
  --database="(default)"

# Create named database (Datastore mode)
gcloud firestore databases create \
  --location=us-east1 \
  --type=DATASTORE_MODE \
  --database=analytics

# List databases
gcloud firestore databases list

# Describe database
gcloud firestore databases describe --database="(default)"

# Delete database (requires delete protection to be disabled)
gcloud firestore databases update --database=staging \
  --delete-protection=DISABLED
gcloud firestore databases delete --database=staging

# ─────────────────────────────────────────────────────────────
# Index Management
# ─────────────────────────────────────────────────────────────

# List indexes
gcloud firestore indexes composite list --database="(default)"

# Create composite index
gcloud firestore indexes composite create \
  --database="(default)" \
  --collection-group=orders \
  --query-scope=COLLECTION \
  --field-config field-path=status,order=ascending \
  --field-config field-path=createdAt,order=descending

# Delete an index
gcloud firestore indexes composite delete INDEX_ID \
  --database="(default)"

# List field overrides (exemptions)
gcloud firestore indexes fields list --database="(default)"

# ─────────────────────────────────────────────────────────────
# Export / Import
# ─────────────────────────────────────────────────────────────

# Export entire database
gcloud firestore export gs://my-backup-bucket/exports/2024-01-15

# Export specific collections
gcloud firestore export gs://my-backup-bucket/exports/2024-01-15 \
  --collection-ids=users,orders

# Import
gcloud firestore import gs://my-backup-bucket/exports/2024-01-15

# ─────────────────────────────────────────────────────────────
# Backup Management
# ─────────────────────────────────────────────────────────────

# Create backup schedule
gcloud firestore backups schedules create \
  --database="(default)" \
  --recurrence=daily \
  --retention=7d

# List backup schedules
gcloud firestore backups schedules list --database="(default)"

# List available backups
gcloud firestore backups list --location=nam5

# Restore from backup
gcloud firestore databases restore \
  --source-backup=projects/my-proj/locations/nam5/backups/2024-01-15 \
  --destination-database=restored-db

# ─────────────────────────────────────────────────────────────
# TTL Management
# ─────────────────────────────────────────────────────────────

# Enable TTL on a field
gcloud firestore fields ttls update expiresAt \
  --collection-group=sessions \
  --enable-ttl \
  --database="(default)"

# List TTL policies
gcloud firestore fields ttls list --database="(default)"
```

---

## Part 15: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: STARTUP (MVP / Small App)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Mobile app with < 10K users                               │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌──────────────┐     ┌──────────────┐             │
│ │ Mobile  │────→│ Firebase SDK │────→│  Firestore   │             │
│ │ App     │     │ (Direct)     │     │  (default)   │             │
│ │         │←────│              │←────│  Native mode │             │
│ └─────────┘     └──────────────┘     └──────────────┘             │
│                                                                       │
│ Setup:                                                              │
│ ├── Single (default) database, Native mode, regional              │
│ ├── Firebase Auth for user authentication                          │
│ ├── Security rules for access control (no backend needed!)        │
│ ├── Real-time listeners for chat / notifications                  │
│ ├── Offline support enabled for mobile                            │
│ └── Free tier covers initial usage                                 │
│                                                                       │
│ Collections:                                                        │
│ ├── /users/{uid} — user profiles                                  │
│ ├── /users/{uid}/messages/{msgId} — user messages                 │
│ ├── /rooms/{roomId} — chat rooms                                   │
│ └── /rooms/{roomId}/messages/{msgId} — room messages              │
│                                                                       │
│ Cost: Free tier → < $10/month for most MVPs                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: MID-SIZE (Growing SaaS)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: SaaS with 100K+ users, backend services                   │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌──────────────┐     ┌──────────────┐             │
│ │ Web App │────→│ Firebase SDK │────→│  Firestore   │             │
│ │         │←────│ (Real-time)  │←────│  (default)   │             │
│ └─────────┘     └──────────────┘     │  Multi-region│             │
│                                       │  nam5        │             │
│ ┌─────────┐     ┌──────────────┐     │              │             │
│ │ Cloud   │────→│ Admin SDK    │────→│              │             │
│ │ Run     │     │ (Server-side)│     │              │             │
│ │ API     │     └──────────────┘     └──────────────┘             │
│ └─────────┘                                 │                      │
│                                              │ Export               │
│                                              ▼                      │
│                                       ┌──────────────┐             │
│                                       │  BigQuery    │             │
│                                       │  (Analytics) │             │
│                                       └──────────────┘             │
│                                                                       │
│ Setup:                                                              │
│ ├── Multi-region (nam5) for high availability                      │
│ ├── Firebase SDK for web (real-time features)                      │
│ ├── Cloud Run backend uses Admin SDK (bypasses security rules)     │
│ ├── Security rules + server-side validation (defense in depth)     │
│ ├── Composite indexes for complex queries                          │
│ ├── TTL policies for sessions and temporary data                   │
│ ├── Daily backups with 14-day retention                            │
│ ├── Export to BigQuery for analytics                                │
│ └── Distributed counters for likes, views, etc.                    │
│                                                                       │
│ Cost: $50-500/month depending on traffic                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 3: ENTERPRISE (Hybrid Architecture)                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Large org, multiple teams, compliance requirements        │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌──────────────┐     ┌──────────────┐             │
│ │ Web/    │────→│ API Gateway  │────→│  Cloud Run   │             │
│ │ Mobile  │     │              │     │  (Backend)   │             │
│ └─────────┘     └──────────────┘     └──────┬───────┘             │
│                                              │                      │
│                           ┌──────────────────┼──────────────────┐  │
│                           │                  │                  │  │
│                           ▼                  ▼                  ▼  │
│                    ┌────────────┐     ┌────────────┐    ┌────────┐│
│                    │ Firestore  │     │ Cloud SQL  │    │ Cloud  ││
│                    │ (user data,│     │ (relational│    │Storage ││
│                    │  sessions) │     │  data)     │    │(files) ││
│                    │ CMEK       │     └────────────┘    └────────┘│
│                    │ PITR       │                                  │
│                    └────────────┘                                  │
│                                                                       │
│ Setup:                                                              │
│ ├── Multiple named databases (prod, staging, per-team)             │
│ ├── CMEK encryption (key in Cloud KMS)                             │
│ ├── PITR enabled for disaster recovery                             │
│ ├── VPC Service Controls perimeter                                 │
│ ├── No direct client access — all through backend APIs             │
│ ├── Admin SDK only (security rules not used — IAM instead)         │
│ ├── Firestore for user-facing data (profiles, preferences)         │
│ ├── Cloud SQL for relational/transactional data                    │
│ ├── Audit logging enabled (Cloud Audit Logs)                       │
│ └── Delete protection enabled on all databases                     │
│                                                                       │
│ Cost: $500-5000+/month                                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIRESTORE QUICK REFERENCE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Modes:                                                              │
│ ├── Native mode: Real-time, offline, Firebase SDK, security rules  │
│ └── Datastore mode: Server-only, Datastore API, legacy compat      │
│                                                                       │
│ Data model: Database → Collections → Documents → Fields            │
│ Document max size: 1 MiB                                            │
│ Max subcollection depth: 100 levels                                 │
│ Max write rate per doc: 1 write/second (sustained)                  │
│                                                                       │
│ Queries:                                                            │
│ ├── Always indexed (no full scans)                                 │
│ ├── Single-field indexes: automatic                                │
│ ├── Composite indexes: manual (click error link to create)         │
│ └── Collection group queries: across all subcollections            │
│                                                                       │
│ Writes:                                                             │
│ ├── set() — create/overwrite       │ add() — auto-ID create       │
│ ├── update() — partial update      │ delete() — remove doc         │
│ ├── Transactions: read-then-write atomic (max 500 ops)             │
│ └── Batched writes: write-only atomic (max 500 ops)                │
│                                                                       │
│ Security:                                                           │
│ ├── Security rules → client SDK access                             │
│ ├── IAM → server SDK / admin access                                │
│ └── CMEK → customer-managed encryption keys                        │
│                                                                       │
│ Backup/Recovery:                                                    │
│ ├── Scheduled backups (daily/weekly, up to 14 weeks)               │
│ ├── PITR (any second, up to 7 days)                                │
│ └── Export/Import (to/from Cloud Storage)                           │
│                                                                       │
│ Key limits:                                                         │
│ ├── 10,000 writes/second per database (soft, can be increased)     │
│ ├── 1,000,000 concurrent connections per database                  │
│ ├── 200 composite indexes per database                             │
│ ├── 500 operations per transaction/batch                           │
│ └── 100 databases per project                                      │
│                                                                       │
│ Pricing: Per read ($0.06/100K) + write ($0.18/100K) + storage      │
│ Free tier: 50K reads + 20K writes + 20K deletes + 1 GiB/day       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Firestore Data

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING FIRESTORE FROM THE CONSOLE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ DELETING DOCUMENTS:                                                 │
│ Console → Firestore → Data tab → Navigate to collection           │
│ 1. Click on the collection name (e.g., "users")                   │
│ 2. Click on the document you want to delete                       │
│ 3. Click the three-dot menu (⋮) next to the document ID          │
│ 4. Select "Delete document"                                        │
│ 5. Confirm deletion                                                │
│ ⚡ Deleting a document does NOT delete its subcollections.         │
│   Subcollection documents become "orphaned" but still exist       │
│   and still cost storage. You must delete them separately.        │
│                                                                       │
│ DELETING COLLECTIONS:                                               │
│ Console → Firestore → Data tab → Navigate to collection           │
│ 1. Click the three-dot menu (⋮) next to the collection name      │
│ 2. Select "Delete collection"                                      │
│ 3. Type the collection ID to confirm                               │
│ ⚠️ This is a BATCH operation — Firestore deletes all documents    │
│    in the collection one by one under the hood.                    │
│ ⚠️ Large collections may take time to fully delete.                │
│ ⚠️ For very large collections (millions of docs), use the         │
│    Firebase CLI or a Cloud Function with batched deletes instead. │
│                                                                       │
│ MANAGING INDEXES:                                                   │
│ Console → Firestore → Indexes tab                                  │
│ 1. View all composite indexes and their status                    │
│    ├── Status: "Building", "Ready", or "Error"                    │
│    └── Building can take minutes to hours for large datasets      │
│ 2. To create a composite index manually:                           │
│    ├── Click "Create index"                                        │
│    ├── Enter Collection ID                                         │
│    ├── Add fields and their sort order (ASC/DESC)                 │
│    ├── Choose query scope: Collection or Collection group         │
│    └── Click "Create"                                              │
│ 3. ⚡ Easier method: Run your query in code → it fails with an    │
│    error that includes a direct link to create the needed index.  │
│    Just click the link! This is the most common approach.         │
│ 4. To delete an index:                                             │
│    ├── Click the three-dot menu (⋮) next to the index            │
│    └── Select "Delete" → confirm                                   │
│                                                                       │
│ EXPORTING / IMPORTING DATA:                                        │
│ Console → Firestore → Import/Export tab                            │
│ 1. To export:                                                      │
│    ├── Click "Export"                                               │
│    ├── Choose a Cloud Storage bucket (e.g., gs://my-backups/)     │
│    ├── Optionally select specific collection IDs                  │
│    └── Click "Start export"                                        │
│ 2. To import:                                                      │
│    ├── Click "Import"                                              │
│    ├── Enter the Cloud Storage path to the export metadata file   │
│    │   (e.g., gs://my-backups/2026-05-22/all_namespaces/...)     │
│    └── Click "Start import"                                        │
│ ⚠️ Import OVERWRITES existing documents with matching IDs.        │
│ ⚠️ Export/Import is for the entire database or specific            │
│    collections — not individual documents.                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## TTL Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           TTL (TIME-TO-LIVE) POLICIES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ TTL automatically deletes documents after they expire.             │
│ You designate a TIMESTAMP field as the TTL field — when the        │
│ timestamp passes, Firestore deletes the document automatically.   │
│                                                                       │
│ How it works:                                                       │
│ ├── You add a timestamp field to your documents (e.g., "expiresAt")│
│ ├── You configure a TTL policy on that field                      │
│ ├── Firestore checks the field and deletes expired documents      │
│ ├── Deletion is NOT instant — may take up to 24 hours after expiry│
│ ├── Deleted documents are billed as normal deletes ($0.02/100K)   │
│ └── Documents without the TTL field are NOT affected              │
│                                                                       │
│ Setting up TTL:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console:                                                      │   │
│ │ 1. Go to Firestore → click the collection (e.g., "sessions")│   │
│ │ 2. Click on the TTL settings (or Indexes → Single field)     │   │
│ │ 3. Add a TTL policy on the field "expiresAt"                 │   │
│ │                                                               │   │
│ │ Terraform:                                                    │   │
│ │ resource "google_firestore_field" "sessions_ttl" {            │   │
│ │   collection = "sessions"                                     │   │
│ │   field      = "expiresAt"                                    │   │
│ │   ttl_config {}                                               │   │
│ │ }                                                             │   │
│ │                                                               │   │
│ │ gcloud CLI:                                                   │   │
│ │ gcloud firestore fields ttls update expiresAt \               │   │
│ │   --collection-group=sessions \                               │   │
│ │   --enable-ttl                                                │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Example — auto-expiring session documents:                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ // When creating a session, set expiry to 24 hours from now  │   │
│ │ await setDoc(doc(db, "sessions", sessionId), {                │   │
│ │   userId: "user_abc",                                         │   │
│ │   token: "abc123...",                                         │   │
│ │   createdAt: serverTimestamp(),                                │   │
│ │   expiresAt: Timestamp.fromDate(                              │   │
│ │     new Date(Date.now() + 24 * 60 * 60 * 1000)               │   │
│ │   )                                                           │   │
│ │ });                                                           │   │
│ │ // Firestore will auto-delete this doc after 24 hours         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Common use cases:                                                   │
│ ├── Session data — expire after 24h or 7 days                     │
│ ├── Temporary tokens / OTPs — expire after 15 minutes             │
│ ├── Shopping carts — expire abandoned carts after 30 days         │
│ ├── Notifications — auto-clean old notifications after 90 days    │
│ ├── Rate limiting records — expire after the window passes        │
│ └── Audit logs — keep for 1 year, then auto-delete                │
│                                                                       │
│ ⚡ TTL is free to configure — you only pay for the delete ops.     │
│ ⚡ If the TTL field is missing or not a timestamp, the document    │
│   is ignored (not deleted).                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Firebase Emulator for Local Development

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIREBASE EMULATOR                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The Firebase Emulator Suite lets you run Firestore LOCALLY for     │
│ development and testing — no cloud costs, no network latency.      │
│                                                                       │
│ Why use the emulator?                                               │
│ ├── FREE — no billing for reads/writes during development         │
│ ├── Fast — no network round-trips to GCP                          │
│ ├── Safe — experiment without touching production data            │
│ ├── Offline — works without internet                               │
│ ├── CI/CD — run integration tests in pipelines                    │
│ └── Security rules testing — validate rules locally               │
│                                                                       │
│ Setup:                                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Install Firebase CLI (requires Node.js)                     │   │
│ │ npm install -g firebase-tools                                 │   │
│ │                                                               │   │
│ │ # Initialize Firebase in your project                         │   │
│ │ firebase init firestore                                       │   │
│ │                                                               │   │
│ │ # Start the emulator suite (includes Firestore)               │   │
│ │ firebase emulators:start                                      │   │
│ │                                                               │   │
│ │ # Or start only the Firestore emulator                        │   │
│ │ firebase emulators:start --only firestore                     │   │
│ │                                                               │   │
│ │ # Emulator UI available at: http://localhost:4000             │   │
│ │ # Firestore emulator runs on: http://localhost:8080           │   │
│ │                                                               │   │
│ │ # Your app connects to the emulator automatically when        │   │
│ │ # you configure it in code:                                   │   │
│ │ # connectFirestoreEmulator(db, 'localhost', 8080);            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ The emulator UI gives you a visual data browser — just like     │
│   the Cloud Console, but running locally.                          │
│ ⚡ Always develop and test against the emulator first. Only        │
│   connect to real Firestore for staging/production.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 27: Bigtable** → `27-bigtable.md`
