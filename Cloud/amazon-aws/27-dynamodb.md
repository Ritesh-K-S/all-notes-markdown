# Chapter 27: DynamoDB

---

## Table of Contents

- [Overview](#overview)
- [Part 1: DynamoDB Fundamentals](#part-1-dynamodb-fundamentals)
- [Part 2: Creating a Table (Full Portal Walkthrough)](#part-2-creating-a-table-full-portal-walkthrough)
- [Part 3: Primary Keys & Data Modeling](#part-3-primary-keys--data-modeling)
- [Part 4: Secondary Indexes (GSI & LSI)](#part-4-secondary-indexes-gsi--lsi)
- [Part 5: Capacity Modes](#part-5-capacity-modes)
- [Part 6: DynamoDB Streams](#part-6-dynamodb-streams)
- [Part 7: DAX (DynamoDB Accelerator)](#part-7-dax-dynamodb-accelerator)
- [Part 8: Global Tables](#part-8-global-tables)
- [Part 9: Backups & PITR](#part-9-backups--pitr)
- [Part 9.5: TTL, Transactions & PartiQL](#part-95-ttl-transactions--partiql)
- [Part 10: Terraform & CLI Examples](#part-10-terraform--cli-examples)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is DynamoDB? Why NoSQL?

Traditional databases (MySQL, PostgreSQL) are **relational** — they store data in rows and columns with strict schemas, and you query them using SQL with complex JOINs. They're great for structured data, but they can struggle at massive scale.

**DynamoDB is a NoSQL database** — think of it as a giant, super-fast filing cabinet:
- Each "item" (row) is a document with key-value pairs
- No need to define all columns upfront — each item can have different attributes
- Designed to handle **millions of requests per second** with single-digit millisecond response times
- **Fully serverless** — no servers to manage, no patching, no capacity planning

**When to use DynamoDB vs RDS:**
- **DynamoDB**: Simple queries (get by key, scan), massive scale, flexible schema, session data, IoT data, gaming leaderboards
- **RDS/Aurora**: Complex queries with JOINs, transactions across tables, reporting, data with strict relationships

**Simple real-world examples:**
- 🎮 A mobile game stores player profiles and scores (millions of players, simple lookups)
- 🛒 An e-commerce cart service (fast read/write, each cart has different items)
- 📱 A social media app stores user sessions and activity feeds
- 🌐 IoT platform storing billions of sensor readings

Amazon DynamoDB is a fully managed, serverless NoSQL database that delivers single-digit millisecond performance at any scale. It handles millions of requests per second with automatic scaling, zero maintenance, and built-in security.

```
What you'll learn:
├── DynamoDB Fundamentals
│   ├── NoSQL concepts (key-value + document)
│   ├── Tables, items, attributes
│   └── Pricing model
├── Creating a Table (Full Portal Walkthrough)
├── Primary Keys & Data Modeling
│   ├── Partition key (hash key)
│   ├── Partition + sort key (composite)
│   └── Single-table design pattern
├── Secondary Indexes (GSI & LSI)
├── Capacity Modes (On-Demand vs Provisioned)
├── DynamoDB Streams (change data capture)
├── DAX (in-memory cache for DynamoDB)
├── Global Tables (multi-region replication)
├── Backups & Point-in-Time Recovery
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: DynamoDB Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           DYNAMODB CORE CONCEPTS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is DynamoDB?                                                    │
│ ├── Fully managed, serverless NoSQL database                      │
│ ├── Key-value AND document data model                             │
│ ├── Single-digit millisecond latency at any scale                │
│ ├── Automatic scaling (millions of requests/sec)                 │
│ ├── Built-in replication across 3 AZs                            │
│ ├── No server provisioning, patching, or management              │
│ └── Pay-per-request or provisioned capacity                      │
│                                                                       │
│ Data model:                                                          │
│ ├── Table: Collection of items (like a SQL table)                │
│ ├── Item: A single record (like a SQL row). Max 400 KB.         │
│ ├── Attribute: A field on an item (like a SQL column)            │
│ ├── Primary Key: Uniquely identifies each item                   │
│ └── No schema enforcement (except for primary key)               │
│                                                                       │
│ DynamoDB vs SQL:                                                     │
│ ┌──────────────────┬─────────────────┬───────────────────────────┐│
│ │ SQL Concept      │ DynamoDB        │ Notes                     ││
│ ├──────────────────┼─────────────────┼───────────────────────────┤│
│ │ Table            │ Table           │ Same concept               ││
│ │ Row              │ Item            │ Max 400 KB                 ││
│ │ Column           │ Attribute       │ Schema-free (flexible)     ││
│ │ Primary Key      │ Partition Key   │ + optional Sort Key        ││
│ │ Index            │ GSI / LSI       │ Global / Local             ││
│ │ Views            │ N/A             │ Use GSI instead            ││
│ │ JOINs            │ N/A             │ Denormalize instead        ││
│ └──────────────────┴─────────────────┴───────────────────────────┘│
│                                                                       │
│ When to use DynamoDB:                                                │
│ ├── Need single-digit ms latency at scale                        │
│ ├── Key-value lookups (user profiles, sessions, carts)          │
│ ├── Serverless architectures (Lambda + API Gateway)             │
│ ├── High write throughput (IoT, gaming leaderboards)            │
│ ├── Simple access patterns (known queries at design time)       │
│ └── ⚠️ NOT for: complex joins, ad-hoc queries, OLAP             │
│                                                                       │
│ Pricing (us-east-1):                                                │
│ ├── On-Demand: $1.25/million write, $0.25/million read          │
│ ├── Provisioned: $0.00065/WCU/hr, $0.00013/RCU/hr              │
│ ├── Storage: $0.25/GB/month                                      │
│ ├── Free tier: 25 WCU + 25 RCU + 25 GB (always free!)          │
│ └── Streams: $0.02/100K read requests                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Table (Full Portal Walkthrough)

```
Console → DynamoDB → Tables → Create table

┌─────────────────────────────────────────────────────────────────┐
│           CREATE TABLE                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Table name: [Orders]                                           │
│ → Case-sensitive, 3-255 characters                             │
│ → Convention: PascalCase (Users, OrderItems, Sessions)        │
│                                                                   │
│ ── Partition key ──                                             │
│ Partition key: [userId]                                       │
│ Type: [String ▼]                                               │
│   ├── String: Most common (IDs, emails, names)               │
│   ├── Number: Numeric values (timestamps, counters)          │
│   └── Binary: Binary data (hashes, encrypted values)         │
│                                                                   │
│ ── Sort key (optional) ──                                      │
│ ☑ Add sort key                                                 │
│ Sort key: [orderDate]                                         │
│ Type: [String ▼]                                               │
│                                                                   │
│ → Partition key only: Each item uniquely identified by PK    │
│ → Partition + Sort key: Items grouped by PK, sorted by SK   │
│ → ⚡ Use composite key for most real-world use cases          │
│                                                                   │
│ ── Table settings ──                                            │
│ ○ Default settings                                              │
│ ● Customize settings                                            │
│                                                                   │
│ ── Table class ──                                               │
│ ● DynamoDB Standard                                             │
│ ○ DynamoDB Standard-IA                                          │
│                                                                   │
│ → Standard: Frequently accessed data. Default.                │
│ → Standard-IA: Infrequently accessed. 60% lower storage cost │
│   but higher read/write costs. Use for: logs, old orders.    │
│                                                                   │
│ ── Read/write capacity settings ──                             │
│                                                                   │
│ Capacity mode:                                                  │
│ ● On-demand                                                     │
│ ○ Provisioned                                                   │
│                                                                   │
│ → On-demand: Pay per request. Auto-scales instantly.          │
│   Best for: unknown traffic, spiky workloads, new apps.       │
│ → Provisioned: Set specific RCU/WCU. Auto-scaling optional. │
│   Best for: predictable traffic, cost optimization.           │
│   Read capacity units (RCU): [5]                              │
│   Write capacity units (WCU): [5]                             │
│   ☑ Auto Scaling (Target utilization: 70%)                   │
│     Min capacity: [1] WCU / [1] RCU                          │
│     Max capacity: [100] WCU / [100] RCU                      │
│                                                                   │
│ ── Secondary indexes ──                                        │
│ Global secondary indexes:                                      │
│   [Add GSI]                                                    │
│   Index name: [OrdersByDate]                                  │
│   Partition key: [orderStatus] (String)                       │
│   Sort key: [orderDate] (String)                              │
│   Projected attributes: ● All ○ Keys only ○ Include          │
│                                                                   │
│ Local secondary indexes:                                       │
│   [Add LSI] (⚠️ Can only be created at table creation time!) │
│   Index name: [OrdersByAmount]                                │
│   Sort key: [totalAmount] (Number)                            │
│   Projected attributes: ● All ○ Keys only ○ Include          │
│                                                                   │
│ ── Encryption ──                                                │
│ ● Owned by Amazon DynamoDB (free, default)                    │
│ ○ AWS managed key (aws/dynamodb)                               │
│ ○ Customer managed key (CMK)                                   │
│                                                                   │
│ → DynamoDB owned: Free, AWS manages everything               │
│ → AWS managed: Free, visible in KMS, CloudTrail logs         │
│ → Customer managed: You control key rotation, policies.      │
│   Required for cross-account access to encrypted tables.     │
│                                                                   │
│ ── Tags ──                                                      │
│ Key: [environment]  Value: [production]                       │
│                                                                   │
│                    [Create table]                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Primary Keys & Data Modeling

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIMARY KEY PATTERNS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Simple Primary Key (Partition Key only)                  │
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ Table: Users                                                    ││
│ │ PK: userId                                                      ││
│ │                                                                  ││
│ │ userId (PK) │ name      │ email              │ age              ││
│ │ ────────────┼───────────┼────────────────────┼──────            ││
│ │ user-001    │ Alice     │ alice@example.com  │ 28               ││
│ │ user-002    │ Bob       │ bob@example.com    │ 35               ││
│ └─────────────────────────────────────────────────────────────────┘│
│ → Each item uniquely identified by partition key alone            │
│ → Use for: User profiles, product catalog, configurations       │
│                                                                       │
│ Pattern 2: Composite Primary Key (Partition + Sort Key)             │
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ Table: Orders                                                   ││
│ │ PK: userId  SK: orderDate#orderId                               ││
│ │                                                                  ││
│ │ userId (PK) │ orderDate#orderId (SK)│ total  │ status           ││
│ │ ────────────┼──────────────────────┼────────┼──────            ││
│ │ user-001    │ 2024-01-15#ord-100   │ $49.99 │ shipped          ││
│ │ user-001    │ 2024-02-20#ord-200   │ $29.99 │ delivered        ││
│ │ user-002    │ 2024-01-10#ord-150   │ $99.99 │ processing       ││
│ └─────────────────────────────────────────────────────────────────┘│
│ → Items grouped by PK, sorted by SK                               │
│ → Query: Get all orders for user-001 (by PK)                    │
│ → Query: Get orders for user-001 after 2024-02 (PK + SK range) │
│                                                                       │
│ Single-Table Design:                                                │
│ ┌─────────────────────────────────────────────────────────────────┐│
│ │ One table stores multiple entity types!                         ││
│ │                                                                  ││
│ │ PK             │ SK              │ data...                      ││
│ │ ───────────────┼─────────────────┼──────────                    ││
│ │ USER#user-001  │ PROFILE         │ name, email, age             ││
│ │ USER#user-001  │ ORDER#ord-100   │ total, status, date          ││
│ │ USER#user-001  │ ORDER#ord-200   │ total, status, date          ││
│ │ USER#user-001  │ ADDR#home       │ street, city, zip            ││
│ │ ORDER#ord-100  │ ITEM#prod-001   │ qty, price                   ││
│ │ ORDER#ord-100  │ ITEM#prod-002   │ qty, price                   ││
│ │ PROD#prod-001  │ DETAILS         │ name, price, category        ││
│ └─────────────────────────────────────────────────────────────────┘│
│ → All related data fetched in a single query                      │
│ → ⚡ Recommended pattern for DynamoDB (avoid multiple tables)     │
│                                                                       │
│ ⚠️ Key rules:                                                       │
│ ├── Partition key determines which partition stores the item     │
│ ├── Distribute data evenly across partitions (avoid hot keys!)  │
│ ├── Bad PK: "country" (few values = hot partitions)             │
│ ├── Good PK: "userId" (high cardinality, even distribution)    │
│ └── Sort key enables range queries within a partition            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Secondary Indexes (GSI & LSI)

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECONDARY INDEXES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GSI (Global Secondary Index):                                       │
│ ├── Different partition key (and optional sort key) from table   │
│ ├── Created at any time (add, modify, delete)                    │
│ ├── Has its OWN provisioned throughput (separate RCU/WCU)       │
│ ├── Eventually consistent reads only                              │
│ ├── Limit: 20 GSIs per table                                     │
│ ├── Spans all partitions (hence "Global")                        │
│ └── ⚡ Most common way to support additional query patterns       │
│                                                                       │
│ LSI (Local Secondary Index):                                        │
│ ├── Same partition key as table, different sort key              │
│ ├── ⚠️ Must be created at table creation (cannot add later!)     │
│ ├── Shares table's provisioned throughput                        │
│ ├── Supports strongly consistent reads                           │
│ ├── Limit: 5 LSIs per table                                     │
│ ├── 10 GB limit per partition key value                          │
│ └── "Local" = same partition as the base table                   │
│                                                                       │
│ ┌──────────────────┬─────────────────┬───────────────────────────┐│
│ │                  │ GSI              │ LSI                       ││
│ ├──────────────────┼─────────────────┼───────────────────────────┤│
│ │ Partition Key    │ Any attribute    │ Same as table              ││
│ │ Sort Key         │ Any attribute    │ Different from table       ││
│ │ Creation         │ Anytime          │ Table creation only        ││
│ │ Throughput       │ Own RCU/WCU      │ Shares with table          ││
│ │ Consistency      │ Eventually only  │ Eventually + Strongly      ││
│ │ Limit            │ 20 per table     │ 5 per table                ││
│ │ Size limit       │ None             │ 10 GB per PK value         ││
│ └──────────────────┴─────────────────┴───────────────────────────┘│
│                                                                       │
│ Projected attributes (what data is copied to the index):           │
│ ├── All: All attributes (most flexible, highest storage cost)   │
│ ├── Keys only: Only PK + SK of table + index                    │
│ └── Include: Specific attributes you choose                      │
│                                                                       │
│ Example GSI:                                                         │
│ Table: Orders (PK: userId, SK: orderDate)                          │
│ GSI: StatusIndex (PK: orderStatus, SK: orderDate)                  │
│ → Query: "Get all 'pending' orders sorted by date"               │
│ → Without GSI: Full table scan (expensive!)                      │
│ → With GSI: Efficient query on StatusIndex                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Capacity Modes

```
┌─────────────────────────────────────────────────────────────────────┐
│           CAPACITY MODES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Read Capacity Unit (RCU):                                           │
│ ├── 1 RCU = 1 strongly consistent read/sec (up to 4 KB)        │
│ ├── 1 RCU = 2 eventually consistent reads/sec (up to 4 KB)     │
│ ├── Items > 4 KB: rounded up (e.g., 7 KB = 2 RCUs)            │
│ └── Transactional read: 2 RCUs per read                         │
│                                                                       │
│ Write Capacity Unit (WCU):                                          │
│ ├── 1 WCU = 1 write/sec (up to 1 KB)                           │
│ ├── Items > 1 KB: rounded up (e.g., 3 KB = 3 WCUs)            │
│ └── Transactional write: 2 WCUs per write                       │
│                                                                       │
│ On-Demand Mode:                                                      │
│ ├── No capacity planning needed                                   │
│ ├── Pay per request ($1.25/million writes, $0.25/million reads) │
│ ├── Scales instantly to any traffic level                        │
│ ├── Best for: unpredictable traffic, new applications           │
│ ├── Can switch to provisioned (once per 24 hours)               │
│ └── ⚡ Start here if unsure                                      │
│                                                                       │
│ Provisioned Mode:                                                    │
│ ├── You specify RCU/WCU                                          │
│ ├── Cheaper for predictable workloads                            │
│ ├── Auto Scaling recommended (set target utilization 70%)       │
│ ├── Reserved Capacity: Up to 77% off with 1/3-year commitment │
│ ├── Burst capacity: 5 minutes of unused capacity               │
│ └── ⚠️ Throttling if you exceed provisioned capacity            │
│                                                                       │
│ Decision:                                                            │
│ ├── "Don't know traffic" → On-Demand                            │
│ ├── "Spiky/unpredictable" → On-Demand                          │
│ ├── "Steady, known traffic" → Provisioned + Auto Scaling       │
│ └── "Steady + cost sensitive" → Provisioned + Reserved         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: DynamoDB Streams

```
Console → DynamoDB → Tables → Select table → Exports and streams → DynamoDB stream details

┌─────────────────────────────────────────────────────────────────┐
│           DYNAMODB STREAMS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Ordered record of every change (insert, modify, delete) │
│ to items in a DynamoDB table. 24-hour retention.               │
│                                                                   │
│ Stream view type:                                               │
│   ○ Key attributes only                                        │
│   ○ New image (item after modification)                        │
│   ○ Old image (item before modification)                       │
│   ● New and old images (both before and after)                 │
│                                                                   │
│ → Key attributes: Only the primary key of changed item       │
│ → New image: Full item as it appears after the change        │
│ → Old image: Full item as it was before the change           │
│ → New and old: Both versions (best for most use cases)       │
│                                                                   │
│ Common consumers:                                               │
│ ├── Lambda: Trigger function on every change                 │
│ ├── Kinesis Data Streams: Higher throughput processing       │
│ └── Custom application (DynamoDB Streams API)                │
│                                                                   │
│ Use cases:                                                      │
│ ├── Replication: Sync changes to another table/service       │
│ ├── Materialized views: Update aggregated data               │
│ ├── Notifications: Send email/SMS on specific changes        │
│ ├── Analytics: Stream changes to analytics pipeline          │
│ ├── Audit: Log all data changes                               │
│ └── Cross-region replication (Global Tables use streams)     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 7: DAX (DynamoDB Accelerator)

```
Console → DynamoDB → DAX → Clusters → Create cluster

┌─────────────────────────────────────────────────────────────────┐
│           CREATE DAX CLUSTER                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Cluster name: [prod-cache]                                     │
│                                                                   │
│ Node type: [dax.r5.large ▼]                                   │
│ → Determines memory/CPU for caching                           │
│                                                                   │
│ Cluster size: [3] nodes                                       │
│ → 1 primary + 2 replicas (across AZs for HA)                 │
│ → ⚡ Use 3+ nodes for production                              │
│                                                                   │
│ Subnet group: [dax-private ▼]                                 │
│ Security group: [sg-dax ▼]                                    │
│ → Allow port 8111 from application security group            │
│                                                                   │
│ IAM role: [dax-service-role ▼]                                │
│ → Needs dynamodb:* permissions on your tables                │
│                                                                   │
│ Encryption at rest: ☑ Enabled                                 │
│ Encryption in transit: ☑ Enabled                              │
│                                                                   │
│ What DAX does:                                                  │
│ ├── In-memory cache for DynamoDB reads                       │
│ ├── Microsecond latency (vs millisecond for DynamoDB)       │
│ ├── Drop-in replacement (same API — just change endpoint)   │
│ ├── Write-through: Writes go to DynamoDB AND cache           │
│ ├── Item cache: Individual GetItem/PutItem results           │
│ ├── Query cache: Query/Scan results                          │
│ └── 10x read performance improvement                          │
│                                                                   │
│ When to use:                                                    │
│ ├── Read-heavy workloads (10:1 read:write or higher)        │
│ ├── Latency-sensitive (gaming, trading, ad-tech)            │
│ ├── Same data read repeatedly (hot items)                    │
│ └── ⚠️ Not for: write-heavy, infrequent reads, strong consist│
│                                                                   │
│ Application change:                                             │
│ // Before (DynamoDB direct):                                   │
│ // client = new DynamoDB.DocumentClient()                     │
│ // After (DAX):                                                │
│ // client = new AmazonDaxClient({endpoints: ['dax://...']})  │
│ → Same API calls — just change the client!                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Global Tables

```
Console → DynamoDB → Tables → Select table → Global tables → Create replica

┌─────────────────────────────────────────────────────────────────┐
│           GLOBAL TABLES                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Multi-region, multi-active replication. Read and write   │
│ to any region. Changes replicated to all regions.              │
│                                                                   │
│ Add replica region: [EU (Ireland) eu-west-1 ▼]                │
│                                                                   │
│ Architecture:                                                   │
│ ┌──────────────┐       ┌──────────────┐                       │
│ │ us-east-1    │ ←───► │ eu-west-1    │                       │
│ │ (read/write) │ <1sec │ (read/write) │                       │
│ └──────────────┘  lag  └──────────────┘                       │
│                                                                   │
│ Key features:                                                   │
│ ├── Multi-active: Write to ANY region                         │
│ ├── Replication lag: Typically <1 second                      │
│ ├── Last writer wins (conflict resolution)                   │
│ ├── Uses DynamoDB Streams (automatically enabled)            │
│ ├── Up to any number of regions                               │
│ ├── Replication cost: per replicated WCU                     │
│ └── ⚡ Table must use on-demand or provisioned with auto-scale│
│                                                                   │
│ Requirements:                                                   │
│ ├── DynamoDB Streams enabled                                  │
│ ├── Same table name in all regions                            │
│ ├── On-demand or provisioned with auto-scaling               │
│ └── KMS encryption (AWS owned or managed key)                │
│                                                                   │
│ Use cases:                                                      │
│ ├── Global applications (serve users from nearest region)    │
│ ├── Disaster recovery (multi-region active-active)           │
│ └── Data locality (comply with data residency requirements)  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Backups & PITR

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKUPS & POINT-IN-TIME RECOVERY                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ On-Demand Backups:                                                   │
│ Console → DynamoDB → Tables → Select → Backups → Create backup    │
│ ├── Full table backup (no performance impact)                    │
│ ├── Retained until you delete them                                │
│ ├── Can restore to same or different region                      │
│ ├── Restore creates a NEW table                                   │
│ └── Use for: Before major changes, archival                      │
│                                                                       │
│ Point-in-Time Recovery (PITR):                                      │
│ Console → DynamoDB → Tables → Select → Backups → Edit PITR       │
│ ├── ☑ Enable point-in-time recovery                              │
│ ├── Continuous backups for last 35 days                           │
│ ├── Restore to any second within the 35-day window              │
│ ├── Additional cost: ~$0.20/GB/month                             │
│ ├── No performance impact                                         │
│ └── ⚡ Enable for all production tables                           │
│                                                                       │
│ Export to S3:                                                        │
│ Console → DynamoDB → Tables → Select → Exports and streams       │
│ → Export to S3 → Configure:                                       │
│   S3 bucket: [my-exports-bucket]                                  │
│   Export format: ● DynamoDB JSON ○ Amazon Ion                    │
│   Time range: Full export or incremental                          │
│   → Use for: Analytics (query with Athena), long-term archive   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9.5: TTL, Transactions & PartiQL

### TTL (Time-To-Live) — Automatic Item Expiration

TTL lets you **automatically delete items** after a specified time. Perfect for session data, temporary tokens, and cache entries — no cleanup Lambda needed.

```
How it works:
1. Choose a TTL attribute (e.g., "expiresAt")
2. Store a Unix timestamp (epoch seconds) as the value
3. DynamoDB automatically deletes items after that time
4. Deleted items appear in DynamoDB Streams (if enabled)

Example item:
{
  "sessionId": "abc-123",
  "userId": "user-42",
  "expiresAt": 1735689600        ← Unix timestamp for 2025-01-01
}

When the current time passes 1735689600, the item is automatically deleted.
```

**Enable TTL:**
```
Console → DynamoDB → Table → Additional settings → Time to Live (TTL)
  TTL attribute: expiresAt → Turn on

CLI:
aws dynamodb update-time-to-live \
  --table-name Sessions \
  --time-to-live-specification "Enabled=true,AttributeName=expiresAt"
```

> ⚡ TTL deletions are **free** — they don't consume WCUs. Items may take up to 48 hours after expiry to actually be deleted (eventually consistent), but they won't appear in queries after expiry.

### Transactions — All-or-Nothing Operations

DynamoDB Transactions let you group up to 100 actions that **all succeed or all fail** — like a bank transfer where debit and credit must both happen.

```
Use cases:
├── Transfer money: Debit account A AND credit account B
├── Place order: Decrement inventory AND create order record
├── User signup: Create user AND create profile AND reserve username
└── Any multi-item operation that must be atomic

Two API calls:
├── TransactWriteItems: Up to 100 Put/Update/Delete/ConditionCheck
└── TransactGetItems: Up to 100 Get operations (consistent read across items)
```

```python
# Example: Transfer $50 from Account A to Account B
import boto3
client = boto3.client('dynamodb')

client.transact_write_items(
    TransactItems=[
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'accountId': {'S': 'A'}},
                'UpdateExpression': 'SET balance = balance - :amount',
                'ConditionExpression': 'balance >= :amount',  # Don't go negative
                'ExpressionAttributeValues': {':amount': {'N': '50'}}
            }
        },
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'accountId': {'S': 'B'}},
                'UpdateExpression': 'SET balance = balance + :amount',
                'ExpressionAttributeValues': {':amount': {'N': '50'}}
            }
        }
    ]
)
```

> ⚠️ Transactions cost **2x the normal WCU/RCU** because DynamoDB uses a two-phase commit protocol internally.

### PartiQL — SQL-Compatible Queries

If you're familiar with SQL, you can query DynamoDB using **PartiQL** instead of the native API:

```sql
-- Select
SELECT * FROM "Users" WHERE "userId" = 'user-42'

-- Insert
INSERT INTO "Users" VALUE {'userId': 'user-43', 'name': 'Alice', 'age': 30}

-- Update
UPDATE "Users" SET "age" = 31 WHERE "userId" = 'user-43'

-- Delete
DELETE FROM "Users" WHERE "userId" = 'user-43'
```

**Use it from:** Console (DynamoDB → Explore items → Run PartiQL), CLI (`aws dynamodb execute-statement`), or any AWS SDK.

> 💡 PartiQL is a convenience layer — it still uses the same capacity units and follows the same key requirements as the native API.

---

## Part 10: Terraform & CLI Examples

```hcl
resource "aws_dynamodb_table" "orders" {
  name         = "Orders"
  billing_mode = "PAY_PER_REQUEST"  # On-demand
  hash_key     = "userId"
  range_key    = "orderDate"

  attribute {
    name = "userId"
    type = "S"
  }

  attribute {
    name = "orderDate"
    type = "S"
  }

  attribute {
    name = "orderStatus"
    type = "S"
  }

  global_secondary_index {
    name            = "StatusIndex"
    hash_key        = "orderStatus"
    range_key       = "orderDate"
    projection_type = "ALL"
  }

  point_in_time_recovery { enabled = true }
  deletion_protection_enabled = true

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  server_side_encryption { enabled = true }

  tags = { Environment = "prod" }
}
```

```bash
# Create table
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=orderDate,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=orderDate,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# Put item
aws dynamodb put-item \
  --table-name Orders \
  --item '{
    "userId": {"S": "user-001"},
    "orderDate": {"S": "2024-01-15"},
    "total": {"N": "49.99"},
    "status": {"S": "shipped"}
  }'

# Query by partition key
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "userId = :uid" \
  --expression-attribute-values '{":uid": {"S": "user-001"}}'

# Query with sort key range
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "userId = :uid AND orderDate > :date" \
  --expression-attribute-values '{
    ":uid": {"S": "user-001"},
    ":date": {"S": "2024-01-01"}
  }'

# Create backup
aws dynamodb create-backup \
  --table-name Orders \
  --backup-name Orders-backup-20240115

# Enable PITR
aws dynamodb update-continuous-backups \
  --table-name Orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

---

## Part 11: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD DYNAMODB PATTERNS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Serverless API                                           │
│ API Gateway → Lambda → DynamoDB                                     │
│ ├── On-demand capacity (scales with traffic)                     │
│ ├── Single-table design (Users + Orders + Products)             │
│ ├── GSIs for additional query patterns                           │
│ └── DAX for read-heavy endpoints                                 │
│                                                                       │
│ Pattern 2: Session Store                                            │
│ ├── PK: sessionId, TTL: expiresAt                                │
│ ├── On-demand (sessions are unpredictable)                       │
│ ├── TTL auto-deletes expired sessions (free!)                   │
│ └── DAX for microsecond session lookups                          │
│                                                                       │
│ Pattern 3: Event Sourcing                                           │
│ ├── PK: entityId, SK: timestamp#eventId                         │
│ ├── DynamoDB Streams → Lambda → materialize views               │
│ ├── Append-only (never update, only insert)                     │
│ └── PITR for full history                                        │
│                                                                       │
│ Pattern 4: Gaming Leaderboard                                       │
│ ├── Table: PK=gameId, SK=score (inverted for ranking)          │
│ ├── GSI: PK=userId (get user's scores across games)            │
│ ├── Write-heavy during games (on-demand mode)                   │
│ └── Global Tables for multi-region tournaments                  │
│                                                                       │
│ Pattern 5: IoT Data Ingestion                                       │
│ ├── PK: deviceId, SK: timestamp                                 │
│ ├── High write throughput (millions of events/second)           │
│ ├── Streams → Kinesis → analytics pipeline                     │
│ ├── TTL: Delete old data automatically                          │
│ └── Export to S3 for long-term analytics                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common DynamoDB Issues

### "ProvisionedThroughputExceededException"

Your table is getting more reads/writes per second than you provisioned.

```
Solutions:
1. Switch to On-Demand capacity mode (pay per request, auto-scales)
   Console → DynamoDB → Table → Additional settings → Capacity mode

2. If using Provisioned: Enable auto-scaling
   Set min/max RCU/WCU and target utilization (70%)

3. Check for hot partitions — one partition key getting all the traffic
   Use CloudWatch Contributor Insights to identify hot keys
   Fix: Redesign partition key to distribute traffic evenly
```

### "Scan is very slow and expensive"

Scan reads EVERY item in the table. On a 10 GB table, that's expensive.

```
Solutions:
1. Use Query instead (requires knowing the partition key)
2. Create a GSI (Global Secondary Index) for your access pattern
3. If you must scan, use parallel scan with Segment/TotalSegments
4. Use FilterExpression to reduce returned data (but it still reads everything)
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using Scan for lookups | Slow, expensive, reads entire table | Use Query with proper key design |
| Poor partition key choice | Hot partitions, throttling | Use high-cardinality keys (userId, orderId) |
| Not using GSIs | Can only query by primary key | Create GSIs for alternate access patterns |
| Forgetting TTL for temp data | Table grows forever, costs increase | Enable TTL on session/cache items |
| Storing large items (>400 KB) | 400 KB item size limit hit | Store large data in S3, reference in DynamoDB |

---

## Quick Reference

```
DynamoDB Quick Reference:
├── Type: Fully managed, serverless NoSQL (key-value + document)
├── Latency: Single-digit millisecond (microsecond with DAX)
├── Primary Key: Partition key (+ optional sort key)
├── Item size: Max 400 KB
├── Indexes: Up to 20 GSI + 5 LSI per table
├── Capacity: On-demand (pay-per-request) or Provisioned
├── Streams: Change data capture, 24-hour retention
├── DAX: In-memory cache (microsecond reads)
├── Global Tables: Multi-region, multi-active replication
├── PITR: Continuous backups, 35-day recovery window
├── TTL: Automatic item expiration (free!)
├── Free tier: 25 WCU + 25 RCU + 25 GB (always free)
└── ⚡ Best for: Key-value lookups, serverless, high-scale
```

---

## What's Next?

In **Chapter 28: ElastiCache**, we'll cover managed in-memory caching with Redis and Memcached — cluster creation, replication, and caching patterns.
