# Chapter 27: Bigtable

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Bigtable Fundamentals](#part-1-bigtable-fundamentals)
- [Part 2: Architecture — How Bigtable Works](#part-2-architecture--how-bigtable-works)
- [Part 3: Instances & Clusters](#part-3-instances--clusters)
- [Part 4: Creating a Bigtable Instance](#part-4-creating-a-bigtable-instance)
- [Part 5: Tables & Column Families](#part-5-tables--column-families)
- [Part 6: Row Key Design](#part-6-row-key-design)
- [Part 7: Reading & Writing Data](#part-7-reading--writing-data)
- [Part 8: Filters & Queries](#part-8-filters--queries)
- [Part 9: Replication](#part-9-replication)
- [Part 10: Performance Tuning](#part-10-performance-tuning)
- [Part 11: Security & IAM](#part-11-security--iam)
- [Part 12: Monitoring & Troubleshooting](#part-12-monitoring--troubleshooting)
- [Part 13: Integrations — BigQuery, Dataflow, HBase](#part-13-integrations--bigquery-dataflow-hbase)
- [Part 14: Terraform & CLI](#part-14-terraform--cli)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Cloud Bigtable is Google's fully managed, wide-column NoSQL database designed for massive analytical and operational workloads. It's the same database that powers Google Search, Maps, Gmail, and YouTube. Bigtable handles petabytes of data with single-digit millisecond latency, processing millions of rows per second. Unlike Firestore (document-oriented, serverless), Bigtable is a provisioned, high-throughput engine for time-series data, IoT, analytics, and machine learning feature stores.

```
┌─────────────────────────────────────────────────────────────────────┐
│ WHAT YOU'LL LEARN                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ├── 1. What is Bigtable and when to use it                         │
│ ├── 2. Architecture: tablets, nodes, Colossus storage              │
│ ├── 3. Instances, clusters, and node scaling                       │
│ ├── 4. Creating an instance (console walkthrough)                  │
│ ├── 5. Tables, column families, and schema design                  │
│ ├── 6. Row key design — the most critical decision                 │
│ ├── 7. Reading and writing data (cbt, client libraries)            │
│ ├── 8. Filters and scan queries                                    │
│ ├── 9. Multi-cluster replication                                   │
│ ├── 10. Performance tuning and benchmarking                        │
│ ├── 11. Security, IAM, and encryption                              │
│ ├── 12. Monitoring and troubleshooting                             │
│ ├── 13. Integrations (BigQuery, Dataflow, HBase)                   │
│ ├── 14. Terraform and gcloud CLI                                   │
│ └── 15. Real-world architecture patterns                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Bigtable Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS BIGTABLE?                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bigtable is a wide-column NoSQL database — NOT a document DB,      │
│ NOT a relational DB, NOT a key-value store.                        │
│                                                                       │
│ Think of it as a massive sorted map:                                │
│   Row Key → { Column Family : { Column Qualifier : Value } }      │
│                                                                       │
│ Key characteristics:                                                │
│ ├── Wide-column store (sparse, distributed, persistent)            │
│ ├── Provisioned — you choose number of nodes (not serverless)      │
│ ├── Single-digit millisecond latency (p99 < 10ms)                  │
│ ├── Petabyte scale (no practical storage limit)                     │
│ ├── Millions of reads/writes per second                            │
│ ├── Automatic storage management (no sharding/partitioning)        │
│ ├── Multi-cluster replication for HA and geo-distribution          │
│ ├── HBase API compatible (drop-in replacement)                     │
│ ├── No secondary indexes (row key is the ONLY index)               │
│ ├── No SQL (no joins, no GROUP BY, no aggregations)                │
│ ├── No transactions across rows                                    │
│ └── Cells are versioned by timestamp (multiple versions per cell)  │
│                                                                       │
│ When to use Bigtable:                                               │
│ ├── Time-series data (metrics, IoT sensor readings, financial)     │
│ ├── High-throughput analytics (ad tech, clickstream)                │
│ ├── Machine learning feature stores                                │
│ ├── Personalization / recommendation engines                       │
│ ├── Graph data (adjacency lists)                                   │
│ ├── Geospatial data at massive scale                               │
│ └── Any workload needing > 10,000 reads/writes per second          │
│                                                                       │
│ When NOT to use Bigtable:                                           │
│ ├── < 1 TB of data → Firestore or Cloud SQL (cheaper)              │
│ ├── Need SQL/joins → Cloud SQL, Spanner, AlloyDB                   │
│ ├── Need secondary indexes → Firestore, Spanner                    │
│ ├── Document-shaped data → Firestore                                │
│ ├── Need transactions → Spanner, Cloud SQL                          │
│ ├── OLAP / warehousing → BigQuery                                   │
│ └── Simple key-value cache → Memorystore (Redis)                   │
│                                                                       │
│ ⚡ Rule of thumb: If you have < 300 GB and < 10K ops/sec,          │
│    Bigtable is overkill. Use Firestore instead.                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

```
┌──────────────────────┬────────────────┬──────────────┬──────────────┐
│ Feature              │ GCP            │ AWS          │ Azure        │
├──────────────────────┼────────────────┼──────────────┼──────────────┤
│ Wide-column NoSQL    │ Bigtable       │ DynamoDB*    │ Cosmos DB    │
│                      │                │              │ (Cassandra)  │
│ HBase compatible     │ ✅ Yes          │ ❌ No (EMR)   │ ❌ No (HDI)   │
│ Provisioned model    │ ✅ Nodes        │ RCU/WCU or   │ RU/s         │
│                      │                │ on-demand    │              │
│ Serverless option    │ ❌ No (Autoscale│ ✅ On-demand  │ ✅ Serverless │
│                      │    nodes)      │              │              │
│ Multi-region repl    │ ✅ Built-in     │ Global Tables│ Multi-region │
│ Time-series native   │ ✅ Designed for │ ⚠️ Possible   │ ⚠️ Possible   │
│ Max throughput       │ Unlimited**    │ Unlimited**  │ Unlimited**  │
│ Latency              │ < 10ms         │ < 10ms       │ < 10ms       │
│ Secondary indexes    │ ❌ No           │ ✅ Yes (GSI)   │ ✅ Yes        │
│ Min cost (approx)    │ ~$500/mo       │ ~$0 (on-dem) │ ~$25/mo      │
│ Storage model        │ Colossus (SSD/ │ S3-backed    │ Log-structured│
│                      │ HDD)           │              │              │
└──────────────────────┴────────────────┴──────────────┴──────────────┘
* DynamoDB is key-value + document, not exactly wide-column
** With enough nodes/capacity
```

### Pricing Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           BIGTABLE PRICING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bigtable is NOT serverless — you pay for provisioned nodes         │
│ whether you use them or not (unlike Firestore).                    │
│                                                                       │
│ ┌───────────────────────┬────────────────────────────────────────┐ │
│ │ Component             │ Price (approximate, us-central1)       │ │
│ ├───────────────────────┼────────────────────────────────────────┤ │
│ │ SSD node              │ $0.65/hour (~$468/month per node)      │ │
│ │ HDD node              │ $0.65/hour (same — node cost is same)  │ │
│ │ SSD storage           │ $0.17/GB/month                         │ │
│ │ HDD storage           │ $0.026/GB/month                        │ │
│ │ Network egress         │ Standard GCP rates                     │ │
│ │ Replication (storage)  │ Storage cost per replica cluster       │ │
│ └───────────────────────┴────────────────────────────────────────┘ │
│                                                                       │
│ Minimum cost:                                                       │
│ ├── Production: 3 nodes minimum = ~$1,404/month (before storage)  │
│ ├── Development: 1 node = ~$468/month (limited features)          │
│ └── No free tier!                                                  │
│                                                                       │
│ ⚡ Bigtable is expensive for small workloads.                       │
│    Break-even vs Firestore: roughly > 1 TB data or               │
│    > 10,000 sustained ops/sec.                                     │
│                                                                       │
│ ⚡ SSD vs HDD storage type is chosen at CLUSTER creation.          │
│    SSD = low latency (recommended). HDD = batch-only workloads.   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture — How Bigtable Works

```
┌─────────────────────────────────────────────────────────────────────┐
│           BIGTABLE ARCHITECTURE                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                        Client Request                               │
│                             │                                       │
│                             ▼                                       │
│                   ┌─────────────────┐                               │
│                   │   Front-end     │ ← Routing layer               │
│                   │   Server Pool   │   (picks correct node)        │
│                   └────────┬────────┘                               │
│                            │                                        │
│              ┌─────────────┼─────────────┐                          │
│              ▼             ▼             ▼                           │
│       ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│       │  Node 1  │  │  Node 2  │  │  Node 3  │  ← Compute layer   │
│       │          │  │          │  │          │     (processing)     │
│       │ Tablet A │  │ Tablet C │  │ Tablet E │                     │
│       │ Tablet B │  │ Tablet D │  │ Tablet F │                     │
│       └──────────┘  └──────────┘  └──────────┘                     │
│              │             │             │                           │
│              └─────────────┼─────────────┘                          │
│                            │                                        │
│                            ▼                                        │
│              ┌──────────────────────────┐                           │
│              │       Colossus           │  ← Storage layer          │
│              │   (Google's distributed  │    (persistent, shared)   │
│              │    file system)          │                           │
│              │                          │                           │
│              │  ┌─────┐ ┌─────┐ ┌─────┐│                          │
│              │  │SST 1│ │SST 2│ │SST 3││  ← SSTables             │
│              │  └─────┘ └─────┘ └─────┘│    (sorted string tables)│
│              └──────────────────────────┘                           │
│                                                                       │
│ Key insight: Compute (nodes) and Storage (Colossus) are SEPARATE.  │
│ ├── Adding a node = more compute, NOT more storage                │
│ ├── Storage scales independently of nodes                          │
│ ├── Rebalancing tablets between nodes = fast (no data movement)   │
│ └── Node failure = reassign tablets to other nodes (seconds)       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Tablets and Data Distribution

```
┌─────────────────────────────────────────────────────────────────────┐
│           TABLETS (Data Partitions)                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ A table is divided into contiguous ranges of row keys called       │
│ TABLETS. Each tablet is assigned to ONE node for processing.       │
│                                                                       │
│ Table: "sensor_readings"                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Row Keys (sorted lexicographically):                          │   │
│ │                                                               │   │
│ │ ┌──────────────────────┐                                     │   │
│ │ │ Tablet A             │ Row keys: "aaa..." to "ddd..."      │   │
│ │ │ → Assigned to Node 1 │                                     │   │
│ │ └──────────────────────┘                                     │   │
│ │ ┌──────────────────────┐                                     │   │
│ │ │ Tablet B             │ Row keys: "ddd..." to "mmm..."      │   │
│ │ │ → Assigned to Node 2 │                                     │   │
│ │ └──────────────────────┘                                     │   │
│ │ ┌──────────────────────┐                                     │   │
│ │ │ Tablet C             │ Row keys: "mmm..." to "zzz..."      │   │
│ │ │ → Assigned to Node 3 │                                     │   │
│ │ └──────────────────────┘                                     │   │
│ │                                                               │   │
│ │ ⚡ Bigtable automatically splits/merges tablets as data      │   │
│ │    grows or shrinks.                                          │   │
│ │ ⚡ Tablets are rebalanced across nodes for even load.        │   │
│ │ ⚡ Each tablet is ~100 MB to ~8 GB.                          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Why this matters for row key design:                                │
│ ├── Row keys sorted lexicographically determine tablet boundaries │
│ ├── Sequential keys (timestamps) = ALL writes to ONE tablet       │
│ │   = ONE node = HOTSPOT 🔥                                       │
│ ├── Well-distributed keys = writes spread across ALL nodes        │
│ │   = full throughput ✅                                           │
│ └── This is why row key design is THE most important decision      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Write Path

```
┌─────────────────────────────────────────────────────────────────────┐
│           WRITE PATH                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Client                                                              │
│   │                                                                 │
│   │ 1. Write request                                                │
│   ▼                                                                 │
│ Front-end Server                                                    │
│   │                                                                 │
│   │ 2. Route to correct node (based on row key → tablet mapping)   │
│   ▼                                                                 │
│ Node                                                                │
│   │                                                                 │
│   │ 3. Write to commit log (in Colossus) → DURABLE                 │
│   │ 4. Write to in-memory buffer (memtable)                        │
│   │ 5. Acknowledge to client → DONE                                │
│   │                                                                 │
│   │ (Later, background)                                             │
│   │ 6. Memtable flushes to SSTable in Colossus                     │
│   │ 7. Compaction merges SSTables for read efficiency              │
│   │                                                                 │
│ ⚡ Write latency ≈ commit log write time (< 5ms SSD)               │
│ ⚡ No WAL in traditional sense — commit log IS the WAL             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Instances & Clusters

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTANCE → CLUSTER → NODE HIERARCHY                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │ INSTANCE: "my-bigtable"                                       │  │
│ │ (Logical container for clusters)                              │  │
│ │                                                               │  │
│ │   ┌─── Cluster: "my-bt-us-central1" ───────────────────┐    │  │
│ │   │ Zone: us-central1-a                                  │    │  │
│ │   │ Storage: SSD                                         │    │  │
│ │   │ Nodes: 3                                             │    │  │
│ │   │ ┌──────┐ ┌──────┐ ┌──────┐                         │    │  │
│ │   │ │Node 1│ │Node 2│ │Node 3│                         │    │  │
│ │   │ └──────┘ └──────┘ └──────┘                         │    │  │
│ │   └──────────────────────────────────────────────────────┘    │  │
│ │                                                               │  │
│ │   ┌─── Cluster: "my-bt-europe-west1" ──────────────────┐    │  │
│ │   │ Zone: europe-west1-b                                 │    │  │
│ │   │ Storage: SSD                                         │    │  │
│ │   │ Nodes: 3                                             │    │  │
│ │   │ ┌──────┐ ┌──────┐ ┌──────┐                         │    │  │
│ │   │ │Node 1│ │Node 2│ │Node 3│                         │    │  │
│ │   │ └──────┘ └──────┘ └──────┘                         │    │  │
│ │   └──────────────────────────────────────────────────────┘    │  │
│ │                                                               │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Hierarchy rules:                                                    │
│ ├── Instance: 1 or more clusters (logical grouping)               │
│ ├── Cluster: Located in ONE zone, has N nodes, ONE storage type   │
│ ├── Node: Processing unit (~10,000 reads/sec or ~10,000 writes/sec│
│ │   at ~1 KB rows, ~10 MB/s throughput)                            │
│ ├── Minimum nodes: 1 (development) or 3 (production)              │
│ ├── Maximum nodes: Quota-dependent (default 30 per cluster)       │
│ └── All clusters in an instance share the same tables/schema       │
│                                                                       │
│ Instance types:                                                     │
│ ┌──────────────────┬──────────────────────┬────────────────────┐   │
│ │ Type             │ Development          │ Production         │   │
│ ├──────────────────┼──────────────────────┼────────────────────┤   │
│ │ Min nodes        │ 1                    │ 3                  │   │
│ │ Replication      │ Single cluster only  │ Multi-cluster ✅    │   │
│ │ SLA              │ No SLA               │ 99.5% (single)    │   │
│ │                  │                      │ 99.99% (replicated)│   │
│ │ Performance      │ Not guaranteed       │ Predictable ✅      │   │
│ │ Autoscaling      │ ❌ No                 │ ✅ Yes              │   │
│ │ Use case         │ Dev/test, learning   │ Production ✅       │   │
│ │ Upgrade to prod? │ ✅ Yes (one-click)    │ —                  │   │
│ └──────────────────┴──────────────────────┴────────────────────┘   │
│                                                                       │
│ ⚠️ Development → Production: one-click upgrade                    │
│ ⚠️ Production → Development: NOT possible                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Autoscaling

```
┌─────────────────────────────────────────────────────────────────────┐
│           NODE AUTOSCALING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Production clusters support autoscaling — Bigtable automatically   │
│ adds/removes nodes based on CPU utilization and storage.           │
│                                                                       │
│ Configuration:                                                      │
│ ├── Min nodes: 3 (minimum for production SLA)                      │
│ ├── Max nodes: 30 (or higher with quota increase)                  │
│ ├── CPU target: 70% (recommended for read-heavy)                   │
│ │              50% (recommended for write-heavy or latency-critical)│
│ └── Storage target: Auto (Bigtable adds nodes if storage per node  │
│     exceeds recommended limits)                                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Nodes  8│                    ┌───────┐                       │   │
│ │        7│                ┌───┤ Peak  │                       │   │
│ │        6│            ┌───┤   │       │                       │   │
│ │        5│        ┌───┤   │   │       ├───┐                   │   │
│ │        4│    ┌───┤   │   │   │       │   ├───┐               │   │
│ │        3│────┤   │   │   │   │       │   │   ├───────        │   │
│ │         └────┴───┴───┴───┴───┴───────┴───┴───┴──────         │   │
│ │          6am  9am 12pm 3pm  6pm      9pm 12am 3am            │   │
│ │                                                               │   │
│ │  ⚡ Scale-up: ~minutes (fast, Bigtable rebalances tablets)   │   │
│ │  ⚡ Scale-down: ~20 minutes (gradual, avoids thrashing)      │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ After adding nodes, performance improves after ~20 minutes     │
│    (tablet rebalancing takes time).                                 │
│ ⚠️ Storage-based scaling: SSD = 5 TB/node max,                    │
│    HDD = 16 TB/node max. If storage exceeds limit,               │
│    nodes are added automatically.                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Creating a Bigtable Instance

### Console Walkthrough

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Bigtable → Create Instance                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Instance Details                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Instance name:  [my-bigtable-prod]                            │   │
│ │ Instance ID:    [my-bigtable-prod] (auto-generated, immutable)│   │
│ │                                                               │   │
│ │ Instance type:                                                │   │
│ │ ○ Production (recommended) — min 3 nodes, SLA, replication   │   │
│ │ ○ Development — 1 node, no SLA, no replication               │   │
│ │                                                               │   │
│ │ → Select "Production" for real workloads ✅                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 2: Storage Type (per cluster, IMMUTABLE after creation)       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ SSD (recommended ✅)                                        │   │
│ │   • Lower latency (< 6ms read, < 3ms write)                 │   │
│ │   • $0.17/GB/month                                           │   │
│ │   • Max 5 TB per node                                        │   │
│ │   • Best for: real-time serving, low-latency queries         │   │
│ │                                                               │   │
│ │ ○ HDD                                                         │   │
│ │   • Higher latency (~20ms reads)                             │   │
│ │   • $0.026/GB/month (6x cheaper)                             │   │
│ │   • Max 16 TB per node                                       │   │
│ │   • Best for: batch analytics, large cold datasets, > 10 TB  │   │
│ │                                                               │   │
│ │ ⚠️ Cannot change storage type after cluster creation!        │   │
│ │ ⚠️ Different clusters in same instance CAN have different    │   │
│ │    storage types (e.g., SSD for serving, HDD for analytics). │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 3: Cluster Configuration                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Cluster ID:    [my-bt-us-central1-a]                          │   │
│ │ Region:        [us-central1]                                  │   │
│ │ Zone:          [us-central1-a]                                │   │
│ │                                                               │   │
│ │ Node scaling:                                                 │   │
│ │ ○ Manual allocation: [3] nodes                               │   │
│ │ ○ Autoscaling ✅:                                             │   │
│ │   • Min nodes: [3]                                           │   │
│ │   • Max nodes: [10]                                          │   │
│ │   • CPU utilization target: [70] %                           │   │
│ │                                                               │   │
│ │ [+ Add Cluster] ← Add additional clusters for replication    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 4: Encryption                                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ Google-managed encryption key (default) ✅                  │   │
│ │ ○ Customer-managed encryption key (CMEK)                      │   │
│ │   → Requires KMS key in same location as cluster             │   │
│ │   → Each cluster can have its own CMEK key                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ [CREATE]                                                             │
│                                                                       │
│ ⚡ Instance creation takes ~2-3 minutes.                            │
│ ⚡ Tables are empty — you create them separately.                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Tables & Column Families

```
┌─────────────────────────────────────────────────────────────────────┐
│           BIGTABLE DATA MODEL                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bigtable has a simple but unique data model:                        │
│                                                                       │
│ Table: "sensor_data"                                                │
│ ┌──────────────┬────────────────────────┬─────────────────────────┐│
│ │              │ Column Family: "info"  │ Column Family: "metrics"││
│ │ Row Key      ├───────────┬────────────┼──────────┬──────────────┤│
│ │              │ info:type │ info:loc   │ metrics: │ metrics:     ││
│ │              │           │            │ temp     │ humidity     ││
│ ├──────────────┼───────────┼────────────┼──────────┼──────────────┤│
│ │ sensor#001#  │ "thermo"  │ "floor-1"  │ 23.5     │ 45.2         ││
│ │ 2024-01-15   │ @ts1      │ @ts1       │ @ts1     │ @ts1         ││
│ │              │           │            │ 24.1     │ 44.8         ││
│ │              │           │            │ @ts2     │ @ts2         ││
│ ├──────────────┼───────────┼────────────┼──────────┼──────────────┤│
│ │ sensor#002#  │ "hygro"   │ "floor-2"  │ 21.0     │              ││
│ │ 2024-01-15   │ @ts1      │ @ts1       │ @ts1     │ (no value)   ││
│ └──────────────┴───────────┴────────────┴──────────┴──────────────┘│
│                                                                       │
│ Terminology:                                                        │
│ ├── Row key: Unique identifier (byte string, max 4 KB)            │
│ │   → The ONLY index. All access patterns depend on this.          │
│ ├── Column family: Group of related columns (defined at schema)    │
│ │   → Must be created BEFORE writing data                          │
│ │   → Keep to < 100 families (ideally 1-10)                       │
│ │   → Name: short letters [a-zA-Z0-9_-], max 16 KB               │
│ ├── Column qualifier: Column name within a family (dynamic!)       │
│ │   → Does NOT need to be predefined — create on write            │
│ │   → Can be any byte string                                      │
│ │   → Billions of unique qualifiers per row is OK                 │
│ ├── Cell: Intersection of row + column family + column qualifier   │
│ │   → Stores raw bytes (no types enforced)                         │
│ │   → Can have MULTIPLE versions (timestamped)                    │
│ └── Timestamp: Each cell version has a timestamp                   │
│     → Auto-assigned or client-provided (microseconds)             │
│     → Used for garbage collection (keep latest N versions or      │
│       versions within a time window)                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Column Family Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│           COLUMN FAMILIES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Column families are defined at the TABLE level (schema).            │
│ Each family has its own garbage collection (GC) policy.             │
│                                                                       │
│ GC Policies — how Bigtable cleans up old cell versions:            │
│                                                                       │
│ 1. Max versions: Keep only the latest N versions                   │
│    ┌──────────────────────────────────────────────────┐             │
│    │ Family: "metrics"                                 │             │
│    │ GC Policy: max_num_versions = 1                   │             │
│    │                                                   │             │
│    │ Before GC:   23.5 @ts3, 24.1 @ts2, 22.8 @ts1    │             │
│    │ After GC:    23.5 @ts3 (only latest kept)        │             │
│    └──────────────────────────────────────────────────┘             │
│                                                                       │
│ 2. Max age: Delete versions older than N                           │
│    ┌──────────────────────────────────────────────────┐             │
│    │ Family: "logs"                                    │             │
│    │ GC Policy: max_age = 7 days                       │             │
│    │                                                   │             │
│    │ Versions older than 7 days are garbage collected. │             │
│    └──────────────────────────────────────────────────┘             │
│                                                                       │
│ 3. Intersection (AND): Both conditions must be true               │
│    max_num_versions = 3 AND max_age = 30 days                      │
│    → Keep up to 3 versions, but only if within 30 days             │
│                                                                       │
│ 4. Union (OR): Either condition triggers GC                        │
│    max_num_versions = 5 OR max_age = 7 days                        │
│    → Delete if > 5 versions OR older than 7 days                   │
│                                                                       │
│ Create table with column families (cbt):                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Create table                                                │   │
│ │ cbt createtable sensor_data                                   │   │
│ │                                                               │   │
│ │ # Create column families                                      │   │
│ │ cbt createfamily sensor_data info                             │   │
│ │ cbt createfamily sensor_data metrics                          │   │
│ │                                                               │   │
│ │ # Set GC policy: keep latest 1 version                       │   │
│ │ cbt setgcpolicy sensor_data metrics maxversions=1             │   │
│ │                                                               │   │
│ │ # Set GC policy: delete older than 7 days                    │   │
│ │ cbt setgcpolicy sensor_data logs maxage=168h                  │   │
│ │                                                               │   │
│ │ # Union policy                                                │   │
│ │ cbt setgcpolicy sensor_data raw \                             │   │
│ │   "maxversions=5 or maxage=168h"                              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ GC runs asynchronously — deleted data may still be             │
│    readable for a short period after GC eligibility.               │
│ ⚠️ GC only removes data during compaction (background process).   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Sparse Data Model

```
┌─────────────────────────────────────────────────────────────────────┐
│           SPARSE = NO WASTED STORAGE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Unlike relational databases, Bigtable doesn't store NULLs.         │
│ Empty cells simply don't exist.                                     │
│                                                                       │
│ Row "sensor#001": info:type="thermo", metrics:temp=23.5            │
│ Row "sensor#002": info:type="hygro", info:loc="floor-2"            │
│                                                                       │
│ sensor#001 has no info:loc → no storage cost for that cell         │
│ sensor#002 has no metrics:temp → no storage cost                   │
│                                                                       │
│ This means:                                                         │
│ ├── Different rows can have completely different columns           │
│ ├── You can add new columns anytime (just write to them)           │
│ ├── No ALTER TABLE needed for new columns                          │
│ ├── Wide rows (10,000+ columns) are perfectly fine                 │
│ └── Think of each row as an independent entity                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Row Key Design

```
┌─────────────────────────────────────────────────────────────────────┐
│           ROW KEY DESIGN — THE MOST IMPORTANT DECISION                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The row key is the ONLY index in Bigtable. There are no secondary  │
│ indexes. Your entire query pattern depends on the row key design.  │
│                                                                       │
│ Row key properties:                                                 │
│ ├── Byte string (max 4 KB, shorter is better)                      │
│ ├── Stored in SORTED order (lexicographic / byte order)            │
│ ├── All reads/scans use row key ranges                             │
│ ├── Cannot be changed after writing (must delete + re-write)       │
│ └── Determines data distribution across nodes/tablets              │
│                                                                       │
│ THE GOLDEN RULE:                                                    │
│ Design your row key for your most common READ patterns,            │
│ while ensuring WRITE distribution across nodes.                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Row Key Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMMON ROW KEY PATTERNS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. ❌ BAD: Timestamp only                                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Row key: "2024-01-15T10:30:00"                                │   │
│ │ Row key: "2024-01-15T10:30:01"                                │   │
│ │ Row key: "2024-01-15T10:30:02"                                │   │
│ │                                                               │   │
│ │ Problem: All new writes go to the SAME tablet (latest range) │   │
│ │ Result: ONE node handles ALL writes = HOTSPOT 🔥              │   │
│ │ The other nodes sit idle = wasted money                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. ✅ GOOD: Entity + Timestamp                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Row key: "sensor-001#2024-01-15T10:30:00"                     │   │
│ │ Row key: "sensor-002#2024-01-15T10:30:00"                     │   │
│ │ Row key: "sensor-001#2024-01-15T10:30:01"                     │   │
│ │ Row key: "sensor-003#2024-01-15T10:30:00"                     │   │
│ │                                                               │   │
│ │ Writes distributed across sensors → across tablets → nodes   │   │
│ │ Scans: "Give me all data for sensor-001" → prefix scan ✅     │   │
│ │ Scans: "sensor-001 between 10:00 and 11:00" → range scan ✅  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. ✅ GOOD: Reversed timestamp (latest data first)                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Row key: "sensor-001#" + (Long.MAX_VALUE - timestamp)         │   │
│ │                                                               │   │
│ │ Why: Bigtable sorts ascending. If you often need "latest      │   │
│ │ readings for sensor X", reversed timestamp puts newest first. │   │
│ │ A prefix scan returns most recent data immediately.           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. ✅ GOOD: Hash prefix (for extreme write distribution)            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Row key: SHA256("user-123")[:4] + "#user-123#2024-01-15"      │   │
│ │                                                               │   │
│ │ Distributes perfectly but LOSES scan ability on the entity.  │   │
│ │ Only use if you always know the exact row key.                │   │
│ │ ⚠️ Avoid for time-series (kills range scans).                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 5. ✅ GOOD: Reversed domain (for URL/domain data)                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Instead of: "www.google.com/search/results"                   │   │
│ │ Use:        "com.google.www/search/results"                   │   │
│ │                                                               │   │
│ │ Groups all google.com pages together for efficient scans.    │   │
│ │ (This is how Google Web Search uses Bigtable!)               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 6. ✅ GOOD: Composite key with fixed-width fields                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Row key: "US#CA#94105#2024-01-15"                             │   │
│ │         country#state#zip#date                                │   │
│ │                                                               │   │
│ │ Enables scans at any prefix level:                            │   │
│ │ • "US" → all US data                                         │   │
│ │ • "US#CA" → all California data                              │   │
│ │ • "US#CA#94105" → specific ZIP code                          │   │
│ │ • "US#CA#94105#2024-01" → ZIP + month                        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Anti-Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           ROW KEY ANTI-PATTERNS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ❌ 1. Sequential numeric IDs: 1, 2, 3, 4, 5...                     │
│    All writes go to one tablet → hotspot                            │
│                                                                       │
│ ❌ 2. Timestamp-first: "2024-01-15T10:30:00#sensor-001"             │
│    All current writes go to the latest timestamp range → hotspot   │
│                                                                       │
│ ❌ 3. Purely random keys (UUID): "a7f2c...", "b3e1d..."             │
│    Good distribution but NO meaningful scan patterns               │
│    Can't query "all data for user X" efficiently                   │
│                                                                       │
│ ❌ 4. Human-readable strings with variable length                    │
│    "Alice", "Bob" → poor distribution (most names start with       │
│    few letters). Use fixed-width hashed prefix instead.            │
│                                                                       │
│ ❌ 5. Frequently updated row keys                                    │
│    Bigtable stores data in immutable SSTables — updates create     │
│    new versions. Frequent updates = write amplification.           │
│                                                                       │
│ ✅ Testing your key design:                                         │
│ ├── Use Key Visualizer (Cloud Console → Bigtable → Key Visualizer)│
│ ├── Shows heatmap of read/write traffic across row key ranges     │
│ ├── Bright spots = hotspots = bad key design                       │
│ └── Available after ~30 minutes of traffic                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Reading & Writing Data

### cbt CLI Tool

```
┌─────────────────────────────────────────────────────────────────────┐
│           cbt — Bigtable Command-Line Tool                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ cbt is the primary CLI for interacting with Bigtable data.         │
│ Install: gcloud components install cbt                              │
│                                                                       │
│ Configuration (~/.cbtrc):                                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ project = my-gcp-project                                      │   │
│ │ instance = my-bigtable-prod                                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ # Write data                                                        │
│ cbt set sensor_data \                                               │
│   "sensor-001#2024-01-15T10:30:00" \                                │
│   info:type=thermo \                                                │
│   info:location=floor-1 \                                           │
│   metrics:temp=23.5 \                                               │
│   metrics:humidity=45.2                                              │
│                                                                       │
│ # Read a single row                                                 │
│ cbt read sensor_data \                                              │
│   prefix="sensor-001#2024-01-15T10:30:00"                           │
│                                                                       │
│ # Read with prefix (all data for sensor-001)                       │
│ cbt read sensor_data prefix="sensor-001#"                           │
│                                                                       │
│ # Read specific row range                                           │
│ cbt read sensor_data \                                              │
│   start="sensor-001#2024-01-15T00:00:00" \                          │
│   end="sensor-001#2024-01-16T00:00:00"                               │
│                                                                       │
│ # Read with column filter                                           │
│ cbt read sensor_data \                                              │
│   prefix="sensor-001#" \                                            │
│   columns="metrics:temp"                                             │
│                                                                       │
│ # Count rows (reads all — expensive!)                              │
│ cbt count sensor_data                                               │
│                                                                       │
│ # Delete a row                                                      │
│ cbt deleterow sensor_data "sensor-001#2024-01-15T10:30:00"          │
│                                                                       │
│ # Delete a column from a row                                        │
│ cbt deletecolumn sensor_data \                                      │
│   "sensor-001#2024-01-15T10:30:00" \                                │
│   metrics:humidity                                                   │
│                                                                       │
│ # List tables                                                       │
│ cbt ls                                                              │
│                                                                       │
│ # Describe table (shows column families + GC policies)             │
│ cbt ls sensor_data                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Client Library (Python)

```
┌─────────────────────────────────────────────────────────────────────┐
│           PYTHON CLIENT LIBRARY                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ pip install google-cloud-bigtable                                   │
│                                                                       │
│ from google.cloud import bigtable                                   │
│ from google.cloud.bigtable import column_family                     │
│ from google.cloud.bigtable import row_filters                       │
│ import datetime                                                     │
│                                                                       │
│ # Connect                                                           │
│ client = bigtable.Client(project="my-gcp-project", admin=True)      │
│ instance = client.instance("my-bigtable-prod")                      │
│ table = instance.table("sensor_data")                               │
│                                                                       │
│ # ── CREATE TABLE ──                                                │
│ if not table.exists():                                              │
│     table.create()                                                  │
│     cf1 = table.column_family("info")                               │
│     cf1.create()                                                    │
│     cf2 = table.column_family(                                      │
│         "metrics",                                                  │
│         gc_rule=column_family.MaxVersionsGCRule(1)                   │
│     )                                                               │
│     cf2.create()                                                    │
│                                                                       │
│ # ── WRITE ──                                                       │
│ row_key = "sensor-001#2024-01-15T10:30:00"                          │
│ row = table.direct_row(row_key)                                     │
│ row.set_cell("info", "type", "thermo")                              │
│ row.set_cell("info", "location", "floor-1")                         │
│ row.set_cell("metrics", "temp", "23.5")                             │
│ row.set_cell("metrics", "humidity", "45.2")                         │
│ row.commit()                                                        │
│                                                                       │
│ # ── BATCH WRITE (mutate_rows) ──                                  │
│ rows = []                                                           │
│ for i in range(1000):                                               │
│     row = table.direct_row(f"sensor-{i:04d}#2024-01-15T10:30:00")  │
│     row.set_cell("metrics", "temp", str(20.0 + i * 0.1))           │
│     rows.append(row)                                                │
│ # Write in batch (max 100,000 mutations per call)                  │
│ table.mutate_rows(rows)                                             │
│                                                                       │
│ # ── READ SINGLE ROW ──                                            │
│ row = table.read_row("sensor-001#2024-01-15T10:30:00")              │
│ if row:                                                             │
│     cell = row.cells["metrics"][b"temp"][0]                         │
│     print(f"Temp: {cell.value.decode()}")  # "23.5"                │
│                                                                       │
│ # ── SCAN (prefix) ──                                              │
│ rows = table.read_rows(                                             │
│     row_set=bigtable.row_set.RowSet()                               │
│       .add_row_range_from_keys(                                     │
│         start_key=b"sensor-001#",                                   │
│         end_key=b"sensor-001$"  # '$' is next char after '#'       │
│     )                                                               │
│ )                                                                   │
│ for row in rows:                                                    │
│     print(row.row_key, row.cells)                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Filters & Queries

```
┌─────────────────────────────────────────────────────────────────────┐
│           BIGTABLE FILTERS (Server-Side)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bigtable doesn't have SQL. Instead, you use FILTERS to narrow      │
│ down results during read/scan operations. Filters run on the       │
│ server, reducing data sent over the network.                       │
│                                                                       │
│ Filter types:                                                       │
│                                                                       │
│ 1. Row key filters:                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Row key regex                                               │   │
│ │ row_filter = row_filters.RowKeyRegexFilter(b"sensor-001.*")   │   │
│ │                                                               │   │
│ │ # Row key prefix (more efficient than regex)                  │   │
│ │ # Use row_set.add_row_range_from_keys() instead               │   │
│ │                                                               │   │
│ │ # Row sample (random sampling — useful for testing)           │   │
│ │ row_filter = row_filters.RowSampleFilter(0.01)  # 1% of rows │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. Column filters:                                                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Specific column family                                      │   │
│ │ row_filter = row_filters.FamilyNameRegexFilter("metrics")     │   │
│ │                                                               │   │
│ │ # Specific column qualifier                                   │   │
│ │ row_filter = row_filters.ColumnQualifierRegexFilter(b"temp")  │   │
│ │                                                               │   │
│ │ # Column range (within a family)                              │   │
│ │ row_filter = row_filters.ColumnRangeFilter(                   │   │
│ │     "metrics",                                                │   │
│ │     start_column=b"humidity",                                 │   │
│ │     end_column=b"temp",                                       │   │
│ │     inclusive_end=True                                         │   │
│ │ )                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. Value filters:                                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Value regex (searches cell values!)                         │   │
│ │ row_filter = row_filters.ValueRegexFilter(b"thermo")          │   │
│ │                                                               │   │
│ │ # Value range                                                 │   │
│ │ row_filter = row_filters.ValueRangeFilter(                    │   │
│ │     start_value=b"20.0",                                      │   │
│ │     end_value=b"30.0"                                         │   │
│ │ )                                                             │   │
│ │ # ⚠️ Values are bytes — comparison is lexicographic!         │   │
│ │ # "9.0" > "23.0" in byte order. Pad numbers if using this.  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. Timestamp filters:                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Only cells within a time range                              │   │
│ │ row_filter = row_filters.TimestampRangeFilter(                │   │
│ │     start=datetime(2024, 1, 15),                              │   │
│ │     end=datetime(2024, 1, 16)                                 │   │
│ │ )                                                             │   │
│ │                                                               │   │
│ │ # Latest N versions per cell                                  │   │
│ │ row_filter = row_filters.CellsColumnLimitFilter(1)            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 5. Composing filters:                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Chain (AND — all must match):                               │   │
│ │ row_filter = row_filters.RowFilterChain(filters=[             │   │
│ │     row_filters.FamilyNameRegexFilter("metrics"),             │   │
│ │     row_filters.ColumnQualifierRegexFilter(b"temp"),          │   │
│ │     row_filters.CellsColumnLimitFilter(1)                     │   │
│ │ ])                                                            │   │
│ │                                                               │   │
│ │ # Interleave (OR — any can match):                            │   │
│ │ row_filter = row_filters.RowFilterUnion(filters=[             │   │
│ │     row_filters.ColumnQualifierRegexFilter(b"temp"),          │   │
│ │     row_filters.ColumnQualifierRegexFilter(b"humidity")       │   │
│ │ ])                                                            │   │
│ │                                                               │   │
│ │ # Condition (IF-THEN-ELSE):                                   │   │
│ │ row_filter = row_filters.ConditionalRowFilter(                │   │
│ │     base_filter=row_filters.ValueRegexFilter(b"alert"),       │   │
│ │     true_filter=row_filters.PassAllFilter(True),              │   │
│ │     false_filter=row_filters.BlockAllFilter(True)             │   │
│ │ )                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Apply filter to a read:                                             │
│ rows = table.read_rows(filter_=row_filter)                         │
│ for row in rows:                                                    │
│     print(row.row_key, row.cells)                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│           MULTI-CLUSTER REPLICATION                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bigtable supports automatic replication between clusters in        │
│ different zones or regions. All clusters are read-write.            │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Instance: "my-bigtable"                                      │   │
│ │                                                               │   │
│ │  ┌─────────────────┐     Replication     ┌─────────────────┐ │   │
│ │  │ Cluster: US      │ ←──────────────── → │ Cluster: EU     │ │   │
│ │  │ us-central1-a    │                     │ europe-west1-b  │ │   │
│ │  │ 3 nodes, SSD     │                     │ 3 nodes, SSD    │ │   │
│ │  │ Read ✅ Write ✅  │                     │ Read ✅ Write ✅ │ │   │
│ │  └─────────────────┘                     └─────────────────┘ │   │
│ │           │                                        │          │   │
│ │           │         ┌─────────────────┐            │          │   │
│ │           └────────→│ Cluster: Asia   │←───────────┘          │   │
│ │                     │ asia-east1-a    │                        │   │
│ │                     │ 3 nodes, SSD    │                        │   │
│ │                     │ Read ✅ Write ✅ │                        │   │
│ │                     └─────────────────┘                        │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Replication properties:                                             │
│ ├── Eventual consistency (NOT strong) — typically < 1 second      │
│ ├── ALL clusters accept reads AND writes (active-active)           │
│ ├── Conflict resolution: last-write-wins (by timestamp)            │
│ ├── Replication latency: seconds (cross-region)                    │
│ ├── Max 4 clusters per instance (configurable)                     │
│ ├── Max 2 clusters per zone                                        │
│ └── Each cluster can have different node count and storage type    │
│                                                                       │
│ Use cases:                                                          │
│ ├── High availability (failover to another cluster)                │
│ ├── Low-latency global reads (serve from nearest cluster)          │
│ ├── Workload isolation (analytics cluster vs serving cluster)      │
│ └── Live migration (add new cluster, remove old cluster)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### App Profiles

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP PROFILES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ App profiles control how your application connects to clusters.    │
│ Think of them as "connection configurations."                      │
│                                                                       │
│ Routing policies:                                                   │
│                                                                       │
│ 1. Multi-cluster routing (default):                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ App profile: "default"                                        │   │
│ │ Routing: multi-cluster → any available cluster               │   │
│ │                                                               │   │
│ │ Client → nearest/available cluster (automatic failover)      │   │
│ │ ✅ High availability                                          │   │
│ │ ⚠️ Eventual consistency between clusters                     │   │
│ │ ⚠️ Single-row transactions NOT supported                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. Single-cluster routing:                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ App profile: "analytics"                                      │   │
│ │ Routing: single-cluster → "my-bt-analytics-cluster"          │   │
│ │                                                               │   │
│ │ Client → always connects to specified cluster                │   │
│ │ ✅ Strong consistency (reads own writes)                      │   │
│ │ ✅ Single-row transactions supported                          │   │
│ │ ❌ No automatic failover (manual only)                        │   │
│ │ ✅ Workload isolation (batch analytics on dedicated cluster)  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Common pattern — workload isolation:                               │
│ ├── App profile "serving" → multi-cluster (HA, low-latency)       │
│ ├── App profile "analytics" → single-cluster (HDD, batch reads)   │
│ └── App profile "export" → single-cluster (dedicated for export)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Performance Tuning

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERFORMANCE EXPECTATIONS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Per-node throughput (approximate, 1 KB rows):                      │
│ ┌──────────────────┬──────────────┬────────────────────────────┐   │
│ │ Operation        │ SSD          │ HDD                        │   │
│ ├──────────────────┼──────────────┼────────────────────────────┤   │
│ │ Reads            │ 10,000/sec   │ 500/sec                    │   │
│ │ Writes           │ 10,000/sec   │ 10,000/sec                 │   │
│ │ Scans            │ 220 MB/sec   │ 180 MB/sec                 │   │
│ │ Read latency     │ < 6ms (p99)  │ ~20ms+                     │   │
│ │ Write latency    │ < 3ms (p99)  │ < 3ms (p99)               │   │
│ └──────────────────┴──────────────┴────────────────────────────┘   │
│                                                                       │
│ ⚡ Linear scaling: 3 nodes = 30K reads/sec, 10 nodes = 100K       │
│    (assuming well-distributed row keys!)                            │
│                                                                       │
│ ⚠️ These numbers assume:                                           │
│ ├── Well-distributed row keys (no hotspots)                        │
│ ├── Cluster has been running 20+ minutes (warm tablets)            │
│ ├── Average row size ~1 KB                                         │
│ └── CPU utilization < 70%                                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Tuning Guidelines

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERFORMANCE TUNING CHECKLIST                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Row key design (most impactful):                                │
│    ├── Check Key Visualizer for hotspots                           │
│    ├── Ensure writes spread across all nodes                       │
│    └── Never use timestamp-first or sequential IDs                 │
│                                                                       │
│ 2. Cluster sizing:                                                  │
│    ├── Keep CPU < 70% for read-heavy workloads                     │
│    ├── Keep CPU < 50% for write-heavy or latency-sensitive         │
│    ├── Storage per node: < 5 TB (SSD), < 16 TB (HDD)              │
│    └── Use autoscaling for variable workloads                      │
│                                                                       │
│ 3. Schema design:                                                   │
│    ├── Fewer column families (1-10, ideally < 5)                   │
│    ├── Don't split related data across families                    │
│    ├── Families with different access patterns can help            │
│    │   (Bigtable can read one family without touching others)     │
│    └── Keep row size reasonable (< 100 MB, ideally < 10 MB)       │
│                                                                       │
│ 4. Client configuration:                                            │
│    ├── Use connection pooling (gRPC channels)                      │
│    ├── Use batch mutations for bulk writes                         │
│    ├── Retry with exponential backoff                               │
│    ├── Use server-side filters (don't fetch + filter client-side)  │
│    └── Keep client in same region as cluster                       │
│                                                                       │
│ 5. Testing:                                                         │
│    ├── Run load test for 10+ minutes (Bigtable improves over time)│
│    ├── Pre-split tables for known large datasets                   │
│    ├── Use at least 300 GB of data for realistic benchmarks        │
│    └── Ramp up traffic gradually (don't spike instantly)           │
│                                                                       │
│ 6. Compaction:                                                      │
│    ├── Automatic, but heavy writes cause temporary read slowdown  │
│    ├── After bulk import, wait for compaction to complete          │
│    └── Monitor "compaction" metrics in Cloud Monitoring            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Security & IAM

```
┌─────────────────────────────────────────────────────────────────────┐
│           BIGTABLE SECURITY                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ IAM Roles:                                                          │
│ ┌──────────────────────────────┬──────────────────────────────────┐│
│ │ Role                         │ Permissions                       ││
│ ├──────────────────────────────┼──────────────────────────────────┤│
│ │ bigtable.admin               │ Full admin: instances, tables,   ││
│ │                              │ clusters, IAM                     ││
│ │ bigtable.user                │ Read/write data, manage tables   ││
│ │                              │ (not instances/clusters)          ││
│ │ bigtable.reader              │ Read data only                    ││
│ │ bigtable.viewer              │ List instances/tables (no data)  ││
│ └──────────────────────────────┴──────────────────────────────────┘│
│                                                                       │
│ Best practices:                                                     │
│ ├── Application → service account with bigtable.user              │
│ ├── Analytics pipeline → service account with bigtable.reader     │
│ ├── Admin operations → human with bigtable.admin                  │
│ └── Use IAM Conditions for time-based or resource-based access     │
│                                                                       │
│ Encryption:                                                         │
│ ├── At rest: Always encrypted (Google-managed key by default)      │
│ ├── CMEK: Customer-managed key via Cloud KMS                       │
│ │   → Set per cluster (each cluster can have different key)       │
│ │   → Cannot change after cluster creation                        │
│ ├── In transit: TLS/gRPC (always encrypted)                        │
│ └── No CSEK (customer-supplied keys) support                       │
│                                                                       │
│ Network security:                                                   │
│ ├── VPC Service Controls: restrict Bigtable access to VPC perimeter│
│ ├── Private Google Access: access Bigtable from VMs without        │
│ │   public IPs                                                     │
│ ├── No public IP endpoint for Bigtable (always uses Google APIs)   │
│ └── Audit Logging: Admin Activity (default), Data Access (opt-in) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Monitoring & Troubleshooting

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING BIGTABLE                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Key metrics to watch:                                               │
│                                                                       │
│ ┌────────────────────────────┬──────────────────────────────────┐  │
│ │ Metric                     │ What it means                     │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ CPU utilization (%)        │ > 70% → add nodes                │  │
│ │ Storage utilization        │ > 70% per node → add nodes       │  │
│ │ Request count              │ Reads/writes per second           │  │
│ │ Error count                │ Failed requests (retry, timeout)  │  │
│ │ Server latency (p50/p99)   │ Milliseconds per operation        │  │
│ │ Rows returned by reads     │ Efficiency of read patterns       │  │
│ │ Disk utilization           │ Compaction pressure               │  │
│ └────────────────────────────┴──────────────────────────────────┘  │
│                                                                       │
│ Monitoring tools:                                                   │
│                                                                       │
│ 1. Cloud Console → Bigtable → Instance → Monitoring Tab           │
│    Built-in dashboards for CPU, storage, latency, throughput       │
│                                                                       │
│ 2. Key Visualizer:                                                 │
│    Console → Bigtable → Instance → Key Visualizer                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Heatmap of activity across row key ranges:                    │   │
│ │                                                               │   │
│ │ Row Keys ↑                                                    │   │
│ │ zzz      │░░░░░░░░░░░░░░░░░░░░░░░░░░░                       │   │
│ │ sensor-9 │░░░░░░░░░░░░░░░░░░░░░░░░░░░                       │   │
│ │ sensor-5 │████████████████ HOTSPOT! ████                     │   │
│ │ sensor-1 │░░░░░░░░░░░░░░░░░░░░░░░░░░░                       │   │
│ │ aaa      │░░░░░░░░░░░░░░░░░░░░░░░░░░░                       │   │
│ │          └──────────────────────────────→ Time                │   │
│ │                                                               │   │
│ │ Bright band = hotspot = bad row key design for those keys    │   │
│ │ Even distribution = good ✅                                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. Cloud Monitoring:                                                │
│    Set up alerts:                                                   │
│    ├── CPU > 70% for 5 minutes → alert                            │
│    ├── Server latency p99 > 100ms → alert                         │
│    ├── Error rate > 1% → alert                                     │
│    └── Storage per node > 4 TB (SSD) → alert                      │
│                                                                       │
│ Troubleshooting common issues:                                     │
│ ┌────────────────────────┬──────────────────────────────────────┐  │
│ │ Symptom                │ Likely cause + fix                    │  │
│ ├────────────────────────┼──────────────────────────────────────┤  │
│ │ High latency           │ CPU > 70% → add nodes               │  │
│ │                        │ Hotspot → fix row key design         │  │
│ │                        │ Just added nodes → wait 20 min       │  │
│ │ Uneven CPU across nodes│ Hotspot → check Key Visualizer      │  │
│ │ Write errors           │ Too many mutations/sec → batch them  │  │
│ │ Read errors            │ Rows too large → reduce row size    │  │
│ │ Stale reads            │ Replication lag (eventual consistency)│  │
│ └────────────────────────┴──────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Integrations — BigQuery, Dataflow, HBase

```
┌─────────────────────────────────────────────────────────────────────┐
│           BIGTABLE INTEGRATIONS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. BigQuery (Federated Queries)                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Query Bigtable data directly from BigQuery — no ETL needed!  │   │
│ │                                                               │   │
│ │ -- Create external table in BigQuery                          │   │
│ │ CREATE EXTERNAL TABLE my_dataset.bt_sensors                   │   │
│ │ OPTIONS (                                                     │   │
│ │   format = 'CLOUD_BIGTABLE',                                 │   │
│ │   uris = ['https://googleapis.com/bigtable/projects/         │   │
│ │     my-proj/instances/my-bt/tables/sensor_data'],             │   │
│ │   bigtable_options = '{"readRowkeyAsString": true,            │   │
│ │     "columnFamilies": [{                                      │   │
│ │       "familyId": "metrics",                                  │   │
│ │       "columns": [                                            │   │
│ │         {"qualifierString": "temp", "type": "FLOAT"},         │   │
│ │         {"qualifierString": "humidity", "type": "FLOAT"}      │   │
│ │       ]                                                       │   │
│ │     }]}'                                                      │   │
│ │ );                                                            │   │
│ │                                                               │   │
│ │ -- Now query with SQL!                                        │   │
│ │ SELECT rowkey, metrics.temp.cell.value AS temperature          │   │
│ │ FROM my_dataset.bt_sensors                                    │   │
│ │ WHERE rowkey LIKE 'sensor-001%';                              │   │
│ │                                                               │   │
│ │ ⚡ Great for ad-hoc analytics without moving data.            │   │
│ │ ⚠️ Performance depends on row key scans (not indexed in BQ). │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. Dataflow (Apache Beam)                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Read/write Bigtable from Dataflow pipelines:                  │   │
│ │                                                               │   │
│ │ # Python (Apache Beam)                                        │   │
│ │ from apache_beam.io.gcp.bigtableio import ReadFromBigtable    │   │
│ │ from apache_beam.io.gcp.bigtableio import WriteToBigtable     │   │
│ │                                                               │   │
│ │ # Read from Bigtable → transform → write somewhere           │   │
│ │ with beam.Pipeline() as p:                                    │   │
│ │     (p                                                        │   │
│ │      | ReadFromBigtable(                                      │   │
│ │          project_id="my-proj",                                │   │
│ │          instance_id="my-bt",                                 │   │
│ │          table_id="sensor_data",                              │   │
│ │          filter_=row_filter                                   │   │
│ │        )                                                      │   │
│ │      | beam.Map(transform_row)                                │   │
│ │      | beam.io.WriteToBigQuery("my_dataset.processed")        │   │
│ │     )                                                         │   │
│ │                                                               │   │
│ │ Use cases:                                                    │   │
│ │ ├── ETL: Bigtable → transform → BigQuery / GCS              │   │
│ │ ├── Real-time: Pub/Sub → transform → Bigtable               │   │
│ │ └── Backfill: GCS CSV → parse → Bigtable                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. HBase Compatibility                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Bigtable is API-compatible with Apache HBase 1.x / 2.x.     │   │
│ │                                                               │   │
│ │ Migration path:                                               │   │
│ │ ├── Existing HBase applications → swap connection config     │   │
│ │ ├── Use hbase-bigtable client JAR (drop-in replacement)      │   │
│ │ ├── Most HBase Java API calls work unchanged                 │   │
│ │ └── Some differences:                                         │   │
│ │     ├── No coprocessors (server-side logic)                  │   │
│ │     ├── No namespace support (use separate instances)        │   │
│ │     └── Some admin commands differ                           │   │
│ │                                                               │   │
│ │ <!-- pom.xml -->                                              │   │
│ │ <dependency>                                                  │   │
│ │   <groupId>com.google.cloud.bigtable</groupId>               │   │
│ │   <artifactId>bigtable-hbase-2.x-shaded</artifactId>         │   │
│ │   <version>2.x.x</version>                                   │   │
│ │ </dependency>                                                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. Other integrations:                                             │
│ ├── Hadoop / MapReduce: BigtableIO connector                      │
│ ├── Spark: bigtable-spark connector                                │
│ ├── Grafana: Bigtable datasource plugin for visualization         │
│ └── Pub/Sub → Cloud Functions → Bigtable (event-driven writes)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Terraform & CLI

### Terraform

```hcl
# ─────────────────────────────────────────────────────────────
# Bigtable Instance with Production Cluster
# ─────────────────────────────────────────────────────────────

resource "google_bigtable_instance" "main" {
  name                = "my-bigtable-prod"
  project             = "my-gcp-project"
  deletion_protection = true

  # Primary cluster
  cluster {
    cluster_id   = "my-bt-us-central1"
    zone         = "us-central1-a"
    storage_type = "SSD"

    autoscaling_config {
      min_nodes  = 3
      max_nodes  = 10
      cpu_target = 70  # Percentage
    }
  }

  # Replica cluster (replication)
  cluster {
    cluster_id   = "my-bt-europe-west1"
    zone         = "europe-west1-b"
    storage_type = "SSD"

    autoscaling_config {
      min_nodes  = 3
      max_nodes  = 10
      cpu_target = 70
    }
  }

  # Optional: HDD cluster for analytics
  cluster {
    cluster_id   = "my-bt-analytics"
    zone         = "us-east1-b"
    storage_type = "HDD"
    num_nodes    = 3  # Manual scaling for analytics
  }

  labels = {
    environment = "production"
    team        = "data-platform"
  }
}

# ─────────────────────────────────────────────────────────────
# Table with Column Families
# ─────────────────────────────────────────────────────────────

resource "google_bigtable_table" "sensor_data" {
  name          = "sensor_data"
  instance_name = google_bigtable_instance.main.name
  project       = "my-gcp-project"

  # Optional: Pre-split for known key distribution
  split_keys = ["sensor-1000", "sensor-2000", "sensor-3000"]

  # Optional: Deletion protection
  deletion_protection = "PROTECTED"

  column_family {
    family = "info"
  }

  column_family {
    family = "metrics"
  }

  column_family {
    family = "raw"
  }
}

# ─────────────────────────────────────────────────────────────
# GC Policies per Column Family
# ─────────────────────────────────────────────────────────────

resource "google_bigtable_gc_policy" "metrics_gc" {
  instance_name = google_bigtable_instance.main.name
  table         = google_bigtable_table.sensor_data.name
  column_family = "metrics"
  project       = "my-gcp-project"

  max_version {
    number = 1  # Keep only latest version
  }
}

resource "google_bigtable_gc_policy" "raw_gc" {
  instance_name = google_bigtable_instance.main.name
  table         = google_bigtable_table.sensor_data.name
  column_family = "raw"
  project       = "my-gcp-project"

  max_age {
    duration = "168h"  # 7 days
  }
}

# Union policy (OR): delete if > 5 versions OR older than 30 days
resource "google_bigtable_gc_policy" "info_gc" {
  instance_name = google_bigtable_instance.main.name
  table         = google_bigtable_table.sensor_data.name
  column_family = "info"
  project       = "my-gcp-project"

  mode = "UNION"

  max_version {
    number = 5
  }

  max_age {
    duration = "720h"  # 30 days
  }
}

# ─────────────────────────────────────────────────────────────
# App Profile (workload isolation)
# ─────────────────────────────────────────────────────────────

resource "google_bigtable_app_profile" "analytics" {
  instance       = google_bigtable_instance.main.name
  app_profile_id = "analytics"
  project        = "my-gcp-project"
  description    = "Analytics workload — single cluster routing"

  single_cluster_routing {
    cluster_id                 = "my-bt-analytics"
    allow_transactional_writes = false
  }

  ignore_warnings = true
}

resource "google_bigtable_app_profile" "serving" {
  instance       = google_bigtable_instance.main.name
  app_profile_id = "serving"
  project        = "my-gcp-project"
  description    = "Serving workload — multi-cluster HA"

  multi_cluster_routing_use_any {
    cluster_ids = ["my-bt-us-central1", "my-bt-europe-west1"]
  }
}

# ─────────────────────────────────────────────────────────────
# IAM Binding
# ─────────────────────────────────────────────────────────────

resource "google_bigtable_instance_iam_member" "app_user" {
  instance = google_bigtable_instance.main.name
  project  = "my-gcp-project"
  role     = "roles/bigtable.user"
  member   = "serviceAccount:my-app@my-gcp-project.iam.gserviceaccount.com"
}
```

### gcloud CLI Reference

```bash
# ─────────────────────────────────────────────────────────────
# Instance Management
# ─────────────────────────────────────────────────────────────

# Create production instance with SSD cluster
gcloud bigtable instances create my-bigtable-prod \
  --display-name="Production Bigtable" \
  --cluster-config=id=my-bt-us-central1,zone=us-central1-a,\
nodes=3,storage-type=SSD

# Create development instance (1 node, no SLA)
gcloud bigtable instances create my-bigtable-dev \
  --display-name="Dev Bigtable" \
  --instance-type=DEVELOPMENT \
  --cluster-config=id=my-bt-dev,zone=us-central1-a,storage-type=SSD

# List instances
gcloud bigtable instances list

# Describe instance
gcloud bigtable instances describe my-bigtable-prod

# Upgrade dev to production
gcloud bigtable instances upgrade my-bigtable-dev

# Delete instance (deletes ALL data!)
gcloud bigtable instances delete my-bigtable-prod

# ─────────────────────────────────────────────────────────────
# Cluster Management
# ─────────────────────────────────────────────────────────────

# Add replica cluster
gcloud bigtable clusters create my-bt-europe-west1 \
  --instance=my-bigtable-prod \
  --zone=europe-west1-b \
  --num-nodes=3 \
  --storage-type=SSD

# Scale cluster manually
gcloud bigtable clusters update my-bt-us-central1 \
  --instance=my-bigtable-prod \
  --num-nodes=5

# Enable autoscaling
gcloud bigtable clusters update my-bt-us-central1 \
  --instance=my-bigtable-prod \
  --autoscaling-min-nodes=3 \
  --autoscaling-max-nodes=10 \
  --autoscaling-cpu-target=70

# List clusters
gcloud bigtable clusters list --instances=my-bigtable-prod

# Delete cluster (keeps data in other clusters)
gcloud bigtable clusters delete my-bt-analytics \
  --instance=my-bigtable-prod

# ─────────────────────────────────────────────────────────────
# Table Management
# ─────────────────────────────────────────────────────────────

# List tables (using cbt)
cbt -instance=my-bigtable-prod ls

# Create table
cbt -instance=my-bigtable-prod createtable sensor_data

# Create column family
cbt -instance=my-bigtable-prod createfamily sensor_data info
cbt -instance=my-bigtable-prod createfamily sensor_data metrics

# Set GC policy
cbt -instance=my-bigtable-prod setgcpolicy sensor_data metrics \
  maxversions=1

# Delete table
cbt -instance=my-bigtable-prod deletetable sensor_data

# ─────────────────────────────────────────────────────────────
# Data Operations (cbt)
# ─────────────────────────────────────────────────────────────

# Write
cbt -instance=my-bigtable-prod set sensor_data \
  "sensor-001#2024-01-15T10:30:00" \
  info:type=thermo metrics:temp=23.5

# Read single row
cbt -instance=my-bigtable-prod lookup sensor_data \
  "sensor-001#2024-01-15T10:30:00"

# Read with prefix
cbt -instance=my-bigtable-prod read sensor_data \
  prefix="sensor-001#"

# Read range
cbt -instance=my-bigtable-prod read sensor_data \
  start="sensor-001#2024-01-15" \
  end="sensor-001#2024-01-16"

# Count rows
cbt -instance=my-bigtable-prod count sensor_data

# ─────────────────────────────────────────────────────────────
# App Profiles
# ─────────────────────────────────────────────────────────────

# List app profiles
gcloud bigtable app-profiles list --instance=my-bigtable-prod

# Create single-cluster routing profile
gcloud bigtable app-profiles create analytics \
  --instance=my-bigtable-prod \
  --route-to=my-bt-analytics \
  --description="Analytics workload"

# Create multi-cluster routing profile
gcloud bigtable app-profiles create serving \
  --instance=my-bigtable-prod \
  --route-any \
  --description="Serving workload"

# ─────────────────────────────────────────────────────────────
# Backups
# ─────────────────────────────────────────────────────────────

# Create backup
gcloud bigtable backups create my-backup \
  --instance=my-bigtable-prod \
  --cluster=my-bt-us-central1 \
  --table=sensor_data \
  --expiration-date=2024-02-15T00:00:00Z

# List backups
gcloud bigtable backups list \
  --instance=my-bigtable-prod \
  --cluster=my-bt-us-central1

# Restore from backup
gcloud bigtable instances tables restore \
  --source-instance=my-bigtable-prod \
  --source-cluster=my-bt-us-central1 \
  --source-backup=my-backup \
  --destination-instance=my-bigtable-prod \
  --destination-table=sensor_data_restored
```

---

## Part 15: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: IOT TIME-SERIES                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: 100,000 IoT sensors sending readings every second        │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌──────────┐     ┌──────────┐     ┌────────────┐ │
│ │ Sensors │────→│ Pub/Sub  │────→│ Dataflow │────→│ Bigtable   │ │
│ │ (IoT)   │     │ (ingest) │     │(transform│     │ (3 nodes,  │ │
│ │         │     │          │     │ + write)  │     │  SSD)      │ │
│ └─────────┘     └──────────┘     └──────────┘     └─────┬──────┘ │
│                                                          │         │
│                                    ┌─────────────────────┘         │
│                                    ▼                                │
│                             ┌────────────┐                         │
│                             │ BigQuery   │ ← Federated query       │
│                             │ (analytics)│   (no ETL needed)       │
│                             └────────────┘                         │
│                                                                       │
│ Row key: "device_type#device_id#reverse_timestamp"                 │
│ Example: "thermo#sensor-0042#9999999999-1705312200"                │
│                                                                       │
│ Column families:                                                    │
│ ├── "d" (data): temp, humidity, pressure — GC: maxversions=1      │
│ └── "m" (metadata): firmware, battery — GC: maxversions=3         │
│                                                                       │
│ Cost: ~$1,500/month (3 nodes + ~2 TB SSD storage)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: AD-TECH / CLICKSTREAM                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Tracking user behavior across millions of sessions       │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌──────────┐     ┌──────────────┐                 │
│ │ Web/App │────→│ Cloud Run│────→│ Bigtable     │                 │
│ │ Events  │     │ (API)    │     │ (10 nodes,   │                 │
│ │         │     │          │     │  SSD,         │                 │
│ │         │     │          │     │  2 clusters)  │                 │
│ └─────────┘     └──────────┘     └──────┬───────┘                 │
│                                          │                          │
│                      ┌───────────────────┼───────────────┐         │
│                      ▼                   ▼               ▼         │
│               ┌────────────┐   ┌──────────────┐  ┌────────────┐   │
│               │ ML Pipeline│   │ BigQuery      │  │ Dashboards │   │
│               │ (features) │   │ (reporting)   │  │ (Grafana)  │   │
│               └────────────┘   └──────────────┘  └────────────┘   │
│                                                                       │
│ Row key: "user_hash#reverse_timestamp"                             │
│ Column family: "e" (events) — qualifiers = event type              │
│ GC: maxage=30d (keep 30 days of clickstream)                       │
│                                                                       │
│ App profiles:                                                       │
│ ├── "serving" → multi-cluster (real-time personalization)          │
│ └── "batch" → single-cluster HDD (ML feature extraction)          │
│                                                                       │
│ Cost: ~$5,000/month (10 nodes × 2 clusters + storage)              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 3: FINANCIAL TIME-SERIES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Stock market tick data, millions of ticks per second     │
│                                                                       │
│ Architecture:                                                       │
│ ┌────────────┐     ┌────────────┐     ┌──────────────────────────┐│
│ │ Market     │────→│ Pub/Sub    │────→│ Bigtable                 ││
│ │ Data Feed  │     │ (ordered)  │     │ 20 nodes, SSD            ││
│ │            │     │            │     │ 3 clusters (multi-region)││
│ │            │     │            │     │ CMEK encryption           ││
│ └────────────┘     └────────────┘     └──────────────────────────┘│
│                                                                       │
│ Row key: "exchange#symbol#reverse_timestamp"                       │
│ Example: "NYSE#GOOGL#9999999999-1705312200123"                     │
│                                                                       │
│ Column families:                                                    │
│ ├── "p" (price): bid, ask, last, volume — GC: maxversions=1       │
│ └── "a" (analytics): vwap, moving_avg — GC: maxage=90d            │
│                                                                       │
│ Key design choices:                                                 │
│ ├── Exchange prefix distributes across exchanges                   │
│ ├── Reversed timestamp = latest prices first in scans              │
│ ├── Microsecond precision in timestamp for ordering                │
│ └── Pre-split by exchange for known distribution                   │
│                                                                       │
│ Cost: ~$10,000/month (20 nodes × 3 clusters + TB of SSD storage)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           BIGTABLE QUICK REFERENCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Architecture: Instance → Cluster(s) → Nodes                       │
│ Data model: Row Key → Column Families → Column Qualifiers → Cells │
│ Storage: Colossus (SSD or HDD, per cluster, immutable choice)      │
│                                                                       │
│ Row key: ONLY index, max 4 KB, sorted lexicographically            │
│ Column family: Pre-defined at schema level (max ~100)              │
│ Column qualifier: Dynamic, created on write (no limit)             │
│ Cell: Versioned by timestamp, stores raw bytes                      │
│ Max row size: ~256 MB (practical limit ~10 MB)                      │
│                                                                       │
│ Performance per node (SSD, ~1 KB rows):                            │
│ ├── 10,000 reads/sec, 10,000 writes/sec                           │
│ ├── 220 MB/s scan throughput                                       │
│ └── < 6ms read latency, < 3ms write latency (p99)                 │
│                                                                       │
│ Scaling:                                                            │
│ ├── Linear — 10 nodes ≈ 100K ops/sec                              │
│ ├── Autoscaling: min/max nodes + CPU target                       │
│ └── Rebalancing after scale: ~20 minutes                           │
│                                                                       │
│ Replication:                                                        │
│ ├── Multi-cluster active-active (all clusters read-write)          │
│ ├── Eventual consistency (< 1 sec typical)                         │
│ ├── Last-write-wins conflict resolution                            │
│ └── Max 4 clusters per instance                                    │
│                                                                       │
│ GC policies: maxversions, maxage, intersection (AND), union (OR)   │
│                                                                       │
│ CLI: cbt (data operations), gcloud bigtable (admin operations)     │
│                                                                       │
│ Minimum cost:                                                       │
│ ├── Development: 1 node ≈ $468/month                              │
│ └── Production: 3 nodes ≈ $1,404/month (before storage)           │
│                                                                       │
│ Key don'ts:                                                         │
│ ├── ❌ Don't use timestamp-first row keys                          │
│ ├── ❌ Don't use sequential numeric IDs                            │
│ ├── ❌ Don't use for < 1 TB data (overkill, use Firestore)        │
│ ├── ❌ Don't expect SQL, joins, or secondary indexes              │
│ └── ❌ Don't benchmark with < 300 GB data or < 20 min runtime     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Bigtable Resources

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETING TABLES FROM CONSOLE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Bigtable → Instances → [your instance] → Tables         │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Tables                                                        │   │
│ │ ┌────────────────────┬──────────────┬──────────────────────┐ │   │
│ │ │ Table Name         │ Families     │ Actions              │ │   │
│ │ ├────────────────────┼──────────────┼──────────────────────┤ │   │
│ │ │ sensor_data        │ info,metrics │ ⋮ → Delete           │ │   │
│ │ │ user_events        │ events       │ ⋮ → Delete           │ │   │
│ │ └────────────────────┴──────────────┴──────────────────────┘ │   │
│ │                                                               │   │
│ │ To delete:                                                    │   │
│ │ 1. Click the ⋮ (three-dot menu) next to the table name      │   │
│ │ 2. Select "Delete table"                                      │   │
│ │ 3. Type the table name to confirm                            │   │
│ │ 4. Click [DELETE]                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ Deletion is PERMANENT — all data in the table is lost.         │
│ ⚠️ If deletion_protection is enabled (Terraform), you must        │
│    disable it first before the Console allows deletion.            │
│ ⚡ Create a backup before deleting if the data might be needed:   │
│    Console → Table → Backups → Create Backup                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Removing Clusters from Console

```
┌─────────────────────────────────────────────────────────────────────┐
│           REMOVING CLUSTERS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Bigtable → Instances → [your instance] → Clusters       │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Clusters                                                      │   │
│ │ ┌───────────────────────┬──────────┬───────┬───────────────┐ │   │
│ │ │ Cluster ID            │ Zone     │ Nodes │ Actions       │ │   │
│ │ ├───────────────────────┼──────────┼───────┼───────────────┤ │   │
│ │ │ my-bt-us-central1     │ us-c1-a  │ 3     │ ⋮ → Edit/Del │ │   │
│ │ │ my-bt-europe-west1    │ eu-w1-b  │ 3     │ ⋮ → Delete   │ │   │
│ │ └───────────────────────┴──────────┴───────┴───────────────┘ │   │
│ │                                                               │   │
│ │ To remove a cluster:                                          │   │
│ │ 1. Click the ⋮ menu next to the cluster                     │   │
│ │ 2. Select "Delete cluster"                                    │   │
│ │ 3. Confirm by typing the cluster ID                          │   │
│ │ 4. Click [DELETE]                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Rules & constraints:                                                │
│ ├── You CANNOT delete the last remaining cluster in an instance   │
│ │   (delete the instance instead — see below)                     │
│ ├── Production instances require at least 1 cluster at all times  │
│ ├── Deleting a cluster removes its REPLICA of the data            │
│ │   (data remains in other clusters)                              │
│ ├── Any app profiles routing to the deleted cluster will fail     │
│ │   → Update app profiles BEFORE deleting a cluster              │
│ └── Storage costs for that cluster stop immediately               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Deleting an Instance

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETING AN ENTIRE INSTANCE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Bigtable → Instances → [your instance]                   │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ Pre-deletion checklist:                                       │   │
│ │ ├── 1. Delete all tables first (or back them up)             │   │
│ │ ├── 2. Verify no app profiles are in use by applications     │   │
│ │ ├── 3. Check IAM — ensure no active service accounts rely   │   │
│ │ │      on this instance                                      │   │
│ │ └── 4. Confirm billing — all costs stop after deletion       │   │
│ │                                                               │   │
│ │ Steps:                                                        │   │
│ │ 1. Navigate to the instance overview page                    │   │
│ │ 2. Click [DELETE INSTANCE] at the top                        │   │
│ │ 3. Type the instance ID to confirm                           │   │
│ │ 4. Click [DELETE]                                             │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ This deletes ALL clusters, ALL tables, and ALL data.            │
│ ⚠️ This action is IRREVERSIBLE — no undo, no recycle bin.         │
│ ⚠️ If deletion_protection is set in Terraform, the Console will   │
│    block deletion — set deletion_protection = false first.        │
│                                                                       │
│ ⚡ Tip: For development instances you no longer need, delete them  │
│    to avoid the ~$468/month per-node cost.                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Using Key Visualizer to Detect Hotspots

```
┌─────────────────────────────────────────────────────────────────────┐
│           KEY VISUALIZER — FINDING HOTSPOTS                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Bigtable → Instances → [your instance] → Key Visualizer │
│                                                                       │
│ Key Visualizer is a built-in heatmap tool that shows read/write    │
│ activity across your row key space over time. It's the #1 tool    │
│ for diagnosing performance problems.                                │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ How to read the heatmap:                                      │   │
│ │                                                               │   │
│ │ Row Keys ↑                                                    │   │
│ │ zzz      │░░░░░░░░░░░░░░░░░░░░░░ (cool = low traffic)       │   │
│ │ sensor-9 │░░░░░░░░░░░░░░░░░░░░░░                             │   │
│ │ sensor-5 │████████████████████ ← BRIGHT = HOTSPOT 🔥         │   │
│ │ sensor-1 │░░░░░░░░░░░░░░░░░░░░░░                             │   │
│ │ aaa      │░░░░░░░░░░░░░░░░░░░░░░                             │   │
│ │          └──────────────────────→ Time                        │   │
│ │                                                               │   │
│ │ Y-axis = row key ranges (sorted)                              │   │
│ │ X-axis = time                                                 │   │
│ │ Color intensity = amount of read/write activity               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ What to look for:                                                   │
│ ┌────────────────────────────────┬──────────────────────────────┐  │
│ │ Pattern                       │ Meaning                       │  │
│ ├────────────────────────────────┼──────────────────────────────┤  │
│ │ Bright horizontal band         │ Hotspot on specific row key  │  │
│ │                                │ range — redesign key ❌       │  │
│ │ Bright band moving upward      │ Monotonically increasing     │  │
│ │                                │ keys (e.g., timestamp-first) │  │
│ │ Even color across all keys     │ Well-distributed ✅           │  │
│ │ Periodic bright bursts         │ Batch jobs hitting same keys │  │
│ │ Single bright row              │ One "hot" row being over-    │  │
│ │                                │ read — consider caching      │  │
│ └────────────────────────────────┴──────────────────────────────┘  │
│                                                                       │
│ Requirements:                                                       │
│ ├── Table must have 30+ minutes of read/write traffic             │
│ ├── Available for all instance types (dev + production)            │
│ └── No additional cost to use                                      │
│                                                                       │
│ ⚡ Always check Key Visualizer BEFORE adding more nodes.            │
│    More nodes won't help if traffic is concentrated on one tablet. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Import/Export

### Deleting Data with cbt

```
┌─────────────────────────────────────────────────────────────────────┐
│           cbt DELETE COMMANDS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Delete an entire table (schema + all data):                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ cbt -instance=my-bigtable-prod deletetable sensor_data        │   │
│ │                                                               │   │
│ │ ⚠️ Removes the table, all column families, and all rows.     │   │
│ │ ⚠️ No confirmation prompt — executes immediately!            │   │
│ │ ⚡ To verify table is gone: cbt -instance=my-bigtable-prod ls │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. Delete a single row:                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ cbt -instance=my-bigtable-prod deleterow sensor_data \        │   │
│ │   "sensor-001#2024-01-15T10:30:00"                            │   │
│ │                                                               │   │
│ │ Removes the entire row (all column families + qualifiers).   │   │
│ │ ⚡ Row key must be exact — no wildcards or prefix matching.   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. Delete a column family (from a table):                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ cbt -instance=my-bigtable-prod deletefamily sensor_data raw   │   │
│ │                                                               │   │
│ │ Removes the column family AND all data stored in it.         │   │
│ │ ⚠️ All rows lose their cells under the "raw" family.         │   │
│ │ ⚡ Other column families (info, metrics) are NOT affected.    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. Delete a single column from a row:                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ cbt -instance=my-bigtable-prod deletecolumn sensor_data \     │   │
│ │   "sensor-001#2024-01-15T10:30:00" \                          │   │
│ │   metrics:humidity                                            │   │
│ │                                                               │   │
│ │ Removes only the "humidity" qualifier from the "metrics"     │   │
│ │ family for that specific row.                                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Summary:                                                            │
│ ┌──────────────────────┬───────────────────────────────────────┐   │
│ │ Command              │ What it removes                       │   │
│ ├──────────────────────┼───────────────────────────────────────┤   │
│ │ cbt deletetable      │ Entire table + all data               │   │
│ │ cbt deleterow        │ One row (all families + qualifiers)   │   │
│ │ cbt deletefamily     │ One column family + its data (all rows│   │
│ │ cbt deletecolumn     │ One column in one row                 │   │
│ └──────────────────────┴───────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Import from CSV

```
┌─────────────────────────────────────────────────────────────────────┐
│           IMPORTING CSV DATA INTO BIGTABLE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bigtable has no built-in CSV import command. The most common       │
│ approaches, from simplest to most scalable:                        │
│                                                                       │
│ ── Method 1: Python script (small datasets, < 1 GB) ──            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ import csv                                                    │   │
│ │ from google.cloud import bigtable                             │   │
│ │                                                               │   │
│ │ client = bigtable.Client(project="my-project", admin=True)    │   │
│ │ instance = client.instance("my-bigtable-prod")                │   │
│ │ table = instance.table("sensor_data")                         │   │
│ │                                                               │   │
│ │ with open("data.csv", "r") as f:                              │   │
│ │     reader = csv.DictReader(f)                                │   │
│ │     rows = []                                                 │   │
│ │     for record in reader:                                     │   │
│ │         row_key = f"{record['sensor_id']}#{record['ts']}"     │   │
│ │         row = table.direct_row(row_key)                       │   │
│ │         row.set_cell("metrics", "temp", record["temp"])       │   │
│ │         row.set_cell("metrics", "humidity", record["hum"])    │   │
│ │         rows.append(row)                                      │   │
│ │                                                               │   │
│ │         # Batch every 500 rows                                │   │
│ │         if len(rows) >= 500:                                  │   │
│ │             table.mutate_rows(rows)                           │   │
│ │             rows = []                                         │   │
│ │                                                               │   │
│ │     if rows:  # Flush remaining                               │   │
│ │         table.mutate_rows(rows)                               │   │
│ │                                                               │   │
│ │ ⚡ Batch size of 500 balances memory and throughput.           │   │
│ │ ⚡ For larger files, use multithreading or Dataflow instead.  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ── Method 2: Dataflow (large datasets, > 1 GB) ──                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Upload CSV to Cloud Storage:                               │   │
│ │    gsutil cp data.csv gs://my-bucket/imports/data.csv         │   │
│ │                                                               │   │
│ │ 2. Use a Dataflow template or custom Beam pipeline:           │   │
│ │                                                               │   │
│ │    # Apache Beam pipeline (Python)                            │   │
│ │    import apache_beam as beam                                 │   │
│ │    from apache_beam.io.gcp.bigtableio import WriteToBigtable  │   │
│ │    from google.cloud.bigtable import row as bt_row            │   │
│ │                                                               │   │
│ │    def csv_to_mutation(line):                                  │   │
│ │        fields = line.split(",")                               │   │
│ │        row_key = f"{fields[0]}#{fields[1]}"                   │   │
│ │        row = bt_row.DirectRow(row_key=row_key.encode())       │   │
│ │        row.set_cell("metrics", "temp", fields[2].encode())    │   │
│ │        return row                                             │   │
│ │                                                               │   │
│ │    with beam.Pipeline() as p:                                 │   │
│ │        (p                                                     │   │
│ │         | beam.io.ReadFromText("gs://my-bucket/imports/*.csv")│   │
│ │         | beam.Map(csv_to_mutation)                           │   │
│ │         | WriteToBigtable(                                    │   │
│ │             project_id="my-project",                          │   │
│ │             instance_id="my-bigtable-prod",                   │   │
│ │             table_id="sensor_data"                            │   │
│ │           )                                                   │   │
│ │        )                                                      │   │
│ │                                                               │   │
│ │ ⚡ Dataflow auto-scales workers for massive imports.           │   │
│ │ ⚡ Can process TBs of CSV data in parallel.                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ── Method 3: cbt (one row at a time, testing only) ──             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Not practical for bulk imports, but useful for testing      │   │
│ │ cbt -instance=my-bigtable-prod set sensor_data \              │   │
│ │   "sensor-001#2024-01-15T10:30:00" \                          │   │
│ │   metrics:temp=23.5 metrics:humidity=45.2                     │   │
│ │                                                               │   │
│ │ # Loop from shell (tiny datasets only):                      │   │
│ │ while IFS=, read -r sensor ts temp hum; do                   │   │
│ │   cbt -instance=my-bigtable-prod set sensor_data \            │   │
│ │     "${sensor}#${ts}" metrics:temp="${temp}" \                 │   │
│ │     metrics:humidity="${hum}"                                  │   │
│ │ done < data.csv                                               │   │
│ │                                                               │   │
│ │ ⚠️ Very slow — one RPC per row. Use Python or Dataflow.      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Choosing the right method:                                          │
│ ┌──────────────────────┬───────────────────────────────────────┐   │
│ │ Dataset size         │ Recommended method                    │   │
│ ├──────────────────────┼───────────────────────────────────────┤   │
│ │ < 10 MB (testing)    │ cbt set (shell loop)                  │   │
│ │ 10 MB – 1 GB         │ Python script with mutate_rows        │   │
│ │ 1 GB – 1 TB+         │ Dataflow (Apache Beam) pipeline       │   │
│ └──────────────────────┴───────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 28: Memorystore** → `28-memorystore.md`
