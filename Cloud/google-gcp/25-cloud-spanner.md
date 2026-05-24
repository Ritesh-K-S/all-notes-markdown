# Chapter 25: Cloud Spanner

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Spanner Fundamentals](#part-1-cloud-spanner-fundamentals)
- [Part 2: Architecture — How Spanner Works](#part-2-architecture--how-spanner-works)
- [Part 3: Instances & Instance Configurations](#part-3-instances--instance-configurations)
- [Part 4: Creating a Spanner Instance](#part-4-creating-a-spanner-instance)
- [Part 5: Databases & Schema Design](#part-5-databases--schema-design)
- [Part 6: Primary Keys & Interleaving](#part-6-primary-keys--interleaving)
- [Part 7: Querying — SQL & Reads](#part-7-querying--sql--reads)
- [Part 8: Transactions](#part-8-transactions)
- [Part 9: Secondary Indexes](#part-9-secondary-indexes)
- [Part 10: Backup & Restore](#part-10-backup--restore)
- [Part 11: Security & IAM](#part-11-security--iam)
- [Part 12: Monitoring & Performance](#part-12-monitoring--performance)
- [Part 13: Terraform & CLI](#part-13-terraform--cli)
- [Part 14: Real-World Patterns](#part-14-real-world-patterns)
- [Quick Reference](#quick-reference)
- [Console Walkthrough: Managing & Deleting Spanner Resources](#console-walkthrough-managing--deleting-spanner-resources)
- [Spanner Emulator for Local Development](#spanner-emulator-for-local-development)
- [Additional gcloud Commands](#additional-gcloud-commands)
- [What's Next?](#whats-next)

---

## Overview

Cloud Spanner is Google's globally distributed, horizontally scalable, strongly consistent relational database. It combines the structure of a relational database with the scale of NoSQL — unlimited rows, automatic sharding, and a 99.999% SLA for multi-region configurations. It's the most expensive GCP database, reserved for workloads that truly need global scale and strong consistency.

```
What you'll learn:
├── Spanner Fundamentals
│   ├── What makes Spanner unique
│   ├── When to use Spanner vs Cloud SQL
│   ├── TrueTime — Google's secret weapon
│   └── Pricing model (processing units & nodes)
├── Architecture
│   ├── Splits (automatic sharding)
│   ├── Paxos consensus (multi-region)
│   ├── TrueTime & external consistency
│   └── Read/write flow
├── Instances & Configurations
│   ├── Regional vs multi-region
│   ├── Processing units vs nodes
│   └── Scaling (add/remove nodes live)
├── Schema Design
│   ├── Interleaved tables (parent-child co-location)
│   ├── Primary key design (avoid hotspots!)
│   ├── Anti-patterns (monotonic keys)
│   └── Schema changes (online, no downtime)
├── Transactions
│   ├── Read-write transactions (strong consistency)
│   ├── Read-only transactions (snapshot reads)
│   └── Stale reads (lower latency)
├── Secondary Indexes
├── Backup & Restore
├── Security & IAM
├── Monitoring & Performance
├── Terraform & CLI
└── Real-world patterns
```

---

## Part 1: Cloud Spanner Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS CLOUD SPANNER?                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Spanner = A relational database that scales horizontally    │
│ like NoSQL while maintaining ACID transactions and strong         │
│ consistency across the globe.                                     │
│                                                                       │
│ The "impossible" combination:                                       │
│ ┌───────────────────┐    ┌──────────────────────────┐              │
│ │ Traditional RDBMS │    │ NoSQL (Bigtable, DynamoDB)│              │
│ │ ✅ SQL             │    │ ❌ Limited SQL             │              │
│ │ ✅ ACID            │    │ ❌ Eventual consistency    │              │
│ │ ✅ Strong consistency  │ ✅ Horizontal scale       │              │
│ │ ❌ Vertical scale only │ ✅ Unlimited data          │              │
│ │ ❌ Single region   │    │ ✅ Multi-region            │              │
│ └───────────────────┘    └──────────────────────────┘              │
│           │                           │                            │
│           └───────────┬───────────────┘                            │
│                       ▼                                            │
│         ┌──────────────────────────┐                               │
│         │ CLOUD SPANNER            │                               │
│         │ ✅ SQL (GoogleSQL/PG)    │                               │
│         │ ✅ ACID transactions     │                               │
│         │ ✅ Strong consistency    │                               │
│         │ ✅ Horizontal scale      │                               │
│         │ ✅ Multi-region          │                               │
│         │ ✅ 99.999% SLA           │                               │
│         │ ❌ VERY expensive        │                               │
│         └──────────────────────────┘                               │
│                                                                       │
│ What makes Spanner special:                                         │
│ ├── GLOBALLY DISTRIBUTED: Data automatically replicated across   │
│ │   regions/continents with strong consistency                  │
│ ├── HORIZONTAL SCALING: Add nodes → more throughput + storage   │
│ │   No limit on data size (petabyte-scale)                     │
│ ├── AUTOMATIC SHARDING: Data split across nodes automatically  │
│ │   You don't manage shards — Spanner does it                  │
│ ├── STRONG CONSISTENCY: Every read sees the latest committed   │
│ │   write, even across regions (serializable isolation)        │
│ ├── SQL: Full SQL support (Google SQL dialect + PostgreSQL)    │
│ ├── ACID: Full transactions across rows, tables, and regions  │
│ ├── ZERO DOWNTIME: Schema changes, scaling, and maintenance   │
│ │   all happen online with zero downtime                       │
│ └── 99.999% SLA: Multi-region — 5.26 minutes downtime/year   │
│     99.99% SLA: Regional — 52.6 minutes downtime/year         │
│                                                                       │
│ No direct AWS or Azure equivalent:                                 │
│ ├── AWS: CockroachDB on EC2, or Amazon Aurora (partial)        │
│ │   Aurora is regional-only, not globally distributed          │
│ ├── Azure: Azure Cosmos DB (multi-model, not true relational)  │
│ │   Cosmos offers strong consistency but is NoSQL-first        │
│ └── Spanner is genuinely unique — built on Google's TrueTime  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### When to Use Spanner vs Cloud SQL

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│                      │ Cloud SQL            │ Cloud Spanner        │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Scale                │ Vertical (bigger VM) │ Horizontal (add      │
│                      │ Max 96 vCPUs         │ nodes, unlimited)    │
│ Max storage          │ 64 TB                │ Unlimited (PB+)      │
│ Multi-region         │ ❌ (cross-region     │ ✅ Built-in, auto    │
│                      │ replica only)        │ replication          │
│ Consistency          │ Strong (single node) │ Strong (global!)     │
│ SLA                  │ 99.95% (HA)          │ 99.999% (multi-reg)  │
│ SQL dialect          │ Standard MySQL/PG/SS │ GoogleSQL or PG      │
│ Schema changes       │ May lock tables      │ Online, zero downtime│
│ Pricing              │ $100-1500/mo         │ $1000+/mo minimum    │
│ Min cost             │ $7/mo (f1-micro)     │ ~$65/mo (100 PU)     │
│ Ecosystem            │ Full MySQL/PG compat │ Limited extensions   │
│ Connection           │ Standard drivers     │ Spanner client lib   │
│ Use when             │ Standard relational  │ Global scale, 5-9s   │
│                      │ workloads            │ availability needed  │
└──────────────────────┴──────────────────────┴──────────────────────┘

⚡ Rule of thumb:
├── Can Cloud SQL handle it?           → Use Cloud SQL (cheaper, simpler)
├── Need > 64 TB or > 96 vCPU?        → Consider Spanner
├── Need multi-region strong consistency? → Spanner
├── Need 99.999% availability?         → Spanner (multi-region)
├── Budget < $1000/month for DB?       → Cloud SQL
└── Global financial/gaming/inventory? → Spanner
```

### Pricing Model

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRICING                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Spanner pricing has THREE components:                               │
│                                                                       │
│ 1. COMPUTE (Processing Units or Nodes):                            │
│    ├── Processing Unit (PU): Smallest unit of compute             │
│    │   1 node = 1000 PU                                           │
│    ├── Min: 100 PU (~$0.90/hour regional)                        │
│    ├── Scale in 100 PU increments (up to 1000 PU)               │
│    ├── Above 1000 PU, scale in 1000 PU (1 node) increments     │
│    ├── Regional: ~$0.90/hour per node ($648/month)              │
│    └── Multi-region: ~$2.70/hour per node ($1,944/month)        │
│                                                                       │
│ 2. STORAGE:                                                         │
│    ├── $0.30/GB/month (regional)                                 │
│    ├── $0.50/GB/month (multi-region)                             │
│    └── Includes replicated storage (no separate replication cost)│
│                                                                       │
│ 3. NETWORK:                                                         │
│    ├── Cross-region reads: Standard egress pricing               │
│    └── Same-region: Free                                          │
│                                                                       │
│ Example costs:                                                      │
│ ├── Minimal: 100 PU regional + 10 GB = ~$68/month              │
│ ├── Small: 1 node regional + 100 GB = ~$678/month              │
│ ├── Medium: 3 nodes regional + 1 TB = ~$2,244/month            │
│ ├── Large: 3 nodes multi-region + 5 TB = ~$8,332/month         │
│ └── Enterprise: 10 nodes multi-region + 50 TB = ~$44,440/month │
│                                                                       │
│ ⚡ Spanner is EXPENSIVE. Make sure you need it!                    │
│   Cloud SQL at $150/month handles most workloads fine.           │
│                                                                       │
│ Committed Use Discounts (CUD):                                      │
│ ├── 1-year commitment: ~20% discount                             │
│ └── 3-year commitment: ~40% discount                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture — How Spanner Works

```
┌─────────────────────────────────────────────────────────────────────┐
│           SPANNER ARCHITECTURE                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Hierarchy:                                                          │
│ Instance → Database(s) → Table(s) → Split(s)                     │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │ SPANNER INSTANCE                                             │    │
│ │                                                               │    │
│ │ ┌───────────────────────┐ ┌───────────────────────┐         │    │
│ │ │ Database: orders_db   │ │ Database: users_db    │         │    │
│ │ │ ┌─────────────────┐  │ │ ┌─────────────────┐  │         │    │
│ │ │ │ Table: Orders   │  │ │ │ Table: Users    │  │         │    │
│ │ │ │ ┌─────┐┌─────┐ │  │ │ │ ┌─────┐┌─────┐ │  │         │    │
│ │ │ │ │Split││Split│ │  │ │ │ │Split││Split│ │  │         │    │
│ │ │ │ │  1  ││  2  │ │  │ │ │ │  1  ││  2  │ │  │         │    │
│ │ │ │ └─────┘└─────┘ │  │ │ │ └─────┘└─────┘ │  │         │    │
│ │ │ └─────────────────┘  │ │ └─────────────────┘  │         │    │
│ │ └───────────────────────┘ └───────────────────────┘         │    │
│ └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ SPLITS (automatic sharding):                                       │
│ ├── Spanner automatically divides table data into "splits"      │
│ ├── Each split = contiguous range of rows (by primary key)     │
│ ├── Splits are distributed across nodes for parallelism        │
│ ├── Splits automatically resize:                                │
│ │   ├── Too big → Spanner splits it into two                  │
│ │   ├── Too small → Spanner may merge splits                  │
│ │   └── Hot split → Spanner moves it to less busy node        │
│ ├── You NEVER manage splits — fully automatic                  │
│ └── ⚡ But your PRIMARY KEY design affects split efficiency!     │
│     Bad keys cause "hotspots" (all writes hit one split)       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### TrueTime — The Secret Weapon

```
┌─────────────────────────────────────────────────────────────────────┐
│           TRUETIME                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How does Spanner guarantee global strong consistency?              │
│ Answer: TrueTime — Google's globally synchronized clock.          │
│                                                                       │
│ Regular clocks:                                                     │
│ ├── Server A's clock: 10:00:00.000                               │
│ ├── Server B's clock: 10:00:00.003  (3ms drift!)               │
│ ├── Who committed first? Can't tell for sure!                  │
│ └── Clock drift = ordering problem = consistency problem       │
│                                                                       │
│ TrueTime:                                                          │
│ ├── API returns: [earliest possible time, latest possible time]│
│ ├── Uses GPS receivers + atomic clocks in every datacenter     │
│ ├── Uncertainty window: typically < 7 milliseconds             │
│ ├── Spanner waits for uncertainty to pass before committing    │
│ │   (called "commit wait")                                     │
│ └── Result: Globally ordered transactions with certainty       │
│                                                                       │
│ What this means for you:                                            │
│ ├── Every transaction gets a globally meaningful timestamp     │
│ ├── If Transaction A commits before B, A's timestamp < B's    │
│ │   GUARANTEED, even across continents                         │
│ ├── You get "external consistency" (linearizability)           │
│ └── The trade-off: ~7ms extra latency per write transaction   │
│     (the "commit wait" for TrueTime uncertainty to pass)      │
│                                                                       │
│ ⚡ TrueTime is ONLY available inside Google's datacenters.        │
│   This is why CockroachDB (open-source Spanner-like) uses HLC  │
│   (Hybrid Logical Clocks) instead — not as accurate but works  │
│   without GPS/atomic clocks.                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Multi-Region Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│           MULTI-REGION REPLICATION                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Regional instance (single region):                                  │
│ ┌────────────────────────────────────────────┐                     │
│ │ asia-south1                                 │                     │
│ │ ┌──────────┐ ┌──────────┐ ┌──────────┐   │                     │
│ │ │ Replica 1│ │ Replica 2│ │ Replica 3│   │                     │
│ │ │ (zone-a) │ │ (zone-b) │ │ (zone-c) │   │                     │
│ │ └──────────┘ └──────────┘ └──────────┘   │                     │
│ │ 3 replicas across 3 zones in one region  │                     │
│ │ SLA: 99.99%                                │                     │
│ └────────────────────────────────────────────┘                     │
│                                                                       │
│ Multi-region instance:                                              │
│ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐             │
│ │ US East       │ │ US Central    │ │ US West       │             │
│ │ (2 replicas)  │ │ (2 replicas)  │ │ (1 witness)   │             │
│ │ Read-write ✅  │ │ Read-write ✅  │ │ Vote only     │             │
│ └───────────────┘ └───────────────┘ └───────────────┘             │
│ 5 replicas across 3 regions                                       │
│ Uses PAXOS consensus for writes (majority vote = 3 of 5)        │
│ SLA: 99.999%                                                      │
│                                                                       │
│ How Paxos consensus works for writes:                              │
│ 1. Client sends write to leader replica                          │
│ 2. Leader proposes write to all replicas                         │
│ 3. Majority (3 of 5) must acknowledge                           │
│ 4. Leader commits and responds to client                        │
│ 5. Remaining replicas catch up asynchronously                   │
│                                                                       │
│ ⚡ Write latency in multi-region = time for majority to respond   │
│   Typically: 10-20ms (within continent) to 100-200ms (global)   │
│   Regional: ~5-10ms write latency                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Instances & Instance Configurations

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTANCE CONFIGURATIONS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When you create a Spanner instance, you choose:                    │
│ 1. Instance configuration (WHERE replicas live)                   │
│ 2. Compute capacity (HOW MUCH processing power)                  │
│                                                                       │
│ Regional configurations:                                            │
│ ├── regional-asia-south1: Mumbai (3 zones)                      │
│ ├── regional-us-central1: Iowa (3 zones)                        │
│ ├── regional-europe-west1: Belgium (3 zones)                    │
│ ├── ... (one per GCP region)                                    │
│ ├── 3 read-write replicas in 3 zones                            │
│ ├── Lower latency for writes (same region)                     │
│ ├── SLA: 99.99%                                                 │
│ └── Lower cost                                                  │
│                                                                       │
│ Multi-region configurations:                                        │
│ ├── nam14: North America (us-central1, us-east4 + witness)     │
│ ├── nam-eur-asia1: US, Europe, Asia (global)                   │
│ ├── eur6: Europe (europe-west1, europe-west4 + witness)        │
│ ├── ... (various multi-region combos)                           │
│ ├── 5 replicas across 3+ regions                               │
│ ├── Higher write latency (cross-region consensus)              │
│ ├── SLA: 99.999%                                                │
│ └── ~3x cost of regional                                       │
│                                                                       │
│ Custom instance configurations (dual-region):                      │
│ ├── You pick exactly which regions host replicas                │
│ ├── Example: asia-south1 + asia-southeast1 + witness           │
│ └── Useful for data residency requirements                     │
│                                                                       │
│ ⚡ Cannot change instance configuration after creation!            │
│   Regional → multi-region: Must create new instance and migrate.│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Processing Units & Nodes

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMPUTE CAPACITY                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Processing Units (PU) = unit of compute capacity.                  │
│ 1 Node = 1000 PU.                                                  │
│                                                                       │
│ ┌───────────────┬────────────┬────────────┬──────────────────┐    │
│ │ Capacity      │ PU         │ Approx.    │ Max Storage      │    │
│ │               │            │ Throughput │                   │    │
│ ├───────────────┼────────────┼────────────┼──────────────────┤    │
│ │ Minimum       │ 100 PU     │ ~300 R/s   │ 409.6 GB         │    │
│ │               │            │ ~100 W/s   │                   │    │
│ │ Small         │ 500 PU     │ ~1.5K R/s  │ 2,048 GB (2 TB)  │    │
│ │               │            │ ~500 W/s   │                   │    │
│ │ 1 Node        │ 1000 PU    │ ~10K R/s   │ 4,096 GB (4 TB)  │    │
│ │               │            │ ~2K W/s    │                   │    │
│ │ 3 Nodes       │ 3000 PU    │ ~30K R/s   │ 12 TB            │    │
│ │               │            │ ~6K W/s    │                   │    │
│ │ 10 Nodes      │ 10000 PU   │ ~100K R/s  │ 40 TB            │    │
│ │               │            │ ~20K W/s   │                   │    │
│ └───────────────┴────────────┴────────────┴──────────────────┘    │
│                                                                       │
│ Scaling:                                                            │
│ ├── < 1000 PU: Scale in increments of 100 PU                   │
│ ├── ≥ 1000 PU: Scale in increments of 1000 PU (1 node)        │
│ ├── Scale up/down with ZERO DOWNTIME ✅                         │
│ ├── Takes effect within minutes                                │
│ └── ⚡ Autoscaler available! Adjusts PU based on CPU/storage   │
│                                                                       │
│ ⚡ Storage limit = 4.096 TB per node (regional)                   │
│   If you have 100 TB of data, you need at least 25 nodes.       │
│   Even if CPU load is low, storage limits minimum nodes.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Creating a Spanner Instance

```
Console → Spanner → Create Instance

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE SPANNER INSTANCE                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance name: [global-orders-db]                                  │
│ Instance ID: [global-orders-db]                                    │
│ ⚡ Cannot change after creation.                                    │
│                                                                       │
│ Configuration:                                                      │
│   ○ Regional                                                       │
│     Region: [asia-south1 (Mumbai)]                                │
│   ○ Multi-region                                                   │
│     Config: [nam-eur-asia1 (North America, Europe, Asia)]        │
│   ○ Custom (dual-region)                                           │
│     Base region + read-only replicas                              │
│                                                                       │
│ Compute capacity:                                                   │
│   ○ Processing units: [1000]  (= 1 node)                         │
│   ○ Nodes: [1]                                                     │
│   ⚡ Start with 100-500 PU for dev/staging                         │
│   ⚡ 1000 PU (1 node) minimum for production                      │
│                                                                       │
│ Autoscaling:                                                        │
│   ☑ Enable autoscaling                                            │
│   Min nodes: [1]                                                   │
│   Max nodes: [5]                                                   │
│   High priority CPU target: [65%]                                 │
│   Storage utilization target: [70%]                               │
│   ⚡ Autoscaler adds/removes nodes based on CPU and storage       │
│                                                                       │
│ Edition:                                                            │
│   ○ Standard                                                       │
│   ○ Enterprise (advanced features) ✅                              │
│                                                                       │
│ Encryption:                                                         │
│   ○ Google-managed encryption key (default)                       │
│   ○ Customer-managed encryption key (CMEK)                        │
│                                                                       │
│ Labels:                                                              │
│   environment: prod                                                │
│   team: platform                                                   │
│                                                                       │
│ [CREATE]                                                             │
│ ⚡ Instance creation takes 1-3 minutes.                             │
│                                                                       │
│ After creation → Create a DATABASE inside the instance:           │
│ Console → Instance → Create Database                              │
│ Database name: [orders]                                            │
│ Database dialect:                                                   │
│   ○ Google Standard SQL ← default, full Spanner features         │
│   ○ PostgreSQL ← PG-compatible syntax (limited features)        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Databases & Schema Design

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCHEMA DESIGN FUNDAMENTALS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Spanner uses SQL DDL with some unique features:                    │
│ ├── No AUTO_INCREMENT (would cause hotspots!)                    │
│ ├── No SERIAL / IDENTITY columns                                │
│ ├── Primary key is REQUIRED on every table                      │
│ ├── INTERLEAVED tables for parent-child co-location             │
│ ├── ARRAY and STRUCT column types supported                     │
│ └── Schema changes are ONLINE (no downtime, no locking)        │
│                                                                       │
│ Creating a table:                                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ CREATE TABLE Users (                                          │  │
│ │   UserId     STRING(36) NOT NULL,  -- UUID as string         │  │
│ │   Email      STRING(255) NOT NULL,                           │  │
│ │   Name       STRING(100),                                     │  │
│ │   CreatedAt  TIMESTAMP NOT NULL                               │  │
│ │       OPTIONS (allow_commit_timestamp = true),               │  │
│ │   Metadata   JSON,                                            │  │
│ │ ) PRIMARY KEY (UserId);                                       │  │
│ │                                                               │  │
│ │ -- ⚡ STRING(N) is required — specify max length              │  │
│ │ -- ⚡ No AUTO_INCREMENT — use UUIDs or hash-based IDs        │  │
│ │ -- ⚡ allow_commit_timestamp lets Spanner set the timestamp  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Supported data types:                                               │
│ ├── BOOL                                                         │
│ ├── INT64 (64-bit integer)                                      │
│ ├── FLOAT32, FLOAT64                                            │
│ ├── NUMERIC (exact decimal, 29 digits + 9 decimal)             │
│ ├── STRING(MAX) or STRING(N)                                    │
│ ├── BYTES(MAX) or BYTES(N)                                      │
│ ├── DATE                                                         │
│ ├── TIMESTAMP                                                    │
│ ├── JSON                                                         │
│ ├── ARRAY<type>                                                  │
│ ├── STRUCT (read-only, in queries)                              │
│ └── PROTO, ENUM (Protocol Buffer types)                         │
│                                                                       │
│ Online schema changes:                                              │
│ ├── ALTER TABLE — adds columns, indexes without downtime       │
│ ├── Long-running operation (may take minutes to hours for       │
│ │   indexes on large tables — backfill runs in background)     │
│ ├── Reads and writes continue during schema change             │
│ └── ⚡ This is HUGE — Cloud SQL/RDS may lock tables!             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Primary Keys & Interleaving

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIMARY KEY DESIGN — CRITICAL!                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️ PRIMARY KEY DESIGN IS THE #1 THING YOU MUST GET RIGHT!           │
│                                                                       │
│ Spanner stores rows ordered by primary key.                        │
│ Adjacent keys → same split → same node.                           │
│ If all new rows go to the same place → HOTSPOT!                   │
│                                                                       │
│ ❌ BAD: Monotonically increasing keys                               │
│ ┌────────────────────────────────────────────────┐                 │
│ │ CREATE TABLE Orders (                           │                 │
│ │   OrderId INT64 NOT NULL,  -- 1, 2, 3, 4, 5...│                 │
│ │   ...                                          │                 │
│ │ ) PRIMARY KEY (OrderId);                        │                 │
│ │                                                 │                 │
│ │ Problem: All new inserts go to the END of the  │                 │
│ │ key range → all writes hit ONE split → hotspot!│                 │
│ │                                                 │                 │
│ │ Also bad:                                       │                 │
│ │ ├── Auto-incrementing IDs (1, 2, 3...)        │                 │
│ │ ├── Timestamp as first key column              │                 │
│ │ └── Date-prefixed keys (2026-05-17-001...)    │                 │
│ └────────────────────────────────────────────────┘                 │
│                                                                       │
│ ✅ GOOD: Distributed keys                                           │
│ ┌────────────────────────────────────────────────┐                 │
│ │ Strategy 1: UUIDs (V4 — random)                │                 │
│ │ CREATE TABLE Orders (                           │                 │
│ │   OrderId STRING(36) NOT NULL,                 │                 │
│ │   -- "a3f2b1c4-..." — random, distributed      │                 │
│ │   ...                                          │                 │
│ │ ) PRIMARY KEY (OrderId);                        │                 │
│ │                                                 │                 │
│ │ Strategy 2: Hash-prefix                         │                 │
│ │ OrderId = SHA256(original_id)[0:8] + original  │                 │
│ │ -- Distributes sequential IDs across splits    │                 │
│ │                                                 │                 │
│ │ Strategy 3: Bit-reverse sequential              │                 │
│ │ Instead of: 1, 2, 3, 4, 5                     │                 │
│ │ Bit-reverse: 1→100...0, 2→010...0, 3→110...0  │                 │
│ │ -- Distributes sequential numbers evenly       │                 │
│ │                                                 │                 │
│ │ Strategy 4: GENERATE_UUID() (GoogleSQL)        │                 │
│ │ INSERT INTO Orders (OrderId, ...)              │                 │
│ │ VALUES (GENERATE_UUID(), ...);                 │                 │
│ └────────────────────────────────────────────────┘                 │
│                                                                       │
│ Composite keys:                                                     │
│ ├── OK: PRIMARY KEY (CustomerId, OrderId)                       │
│ │   Rows for same customer are co-located → good for queries   │
│ │   But ensure CustomerId is well-distributed                  │
│ └── Bad: PRIMARY KEY (Timestamp, EventId) ← timestamp hotspot  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Interleaved Tables (Parent-Child Co-location)

```
┌─────────────────────────────────────────────────────────────────────┐
│           INTERLEAVED TABLES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Store child rows physically next to their parent row.       │
│       Joins between parent and child are FAST (same split, no     │
│       network hop).                                                │
│                                                                       │
│ Without interleaving (separate tables):                             │
│ ┌────────────┐         ┌──────────────────┐                        │
│ │ Customers  │         │ Orders            │                        │
│ │ (Split 1)  │         │ (Split 1)         │                        │
│ │ ┌────────┐ │         │ ┌──────────────┐ │                        │
│ │ │ Cust A │ │         │ │ Order 100    │ │  ← Cust A's order,    │
│ │ │ Cust B │ │         │ │ Order 200    │ │    maybe on different │
│ │ │ Cust C │ │         │ │ Order 300    │ │    node!              │
│ │ └────────┘ │         │ └──────────────┘ │                        │
│ └────────────┘         └──────────────────┘                        │
│ ⚡ Joining Customers + Orders may cross nodes = slow             │
│                                                                       │
│ WITH interleaving:                                                  │
│ ┌────────────────────────────────────────┐                         │
│ │ Split 1 (physically together!)          │                         │
│ │ ┌──────────────────────────────┐       │                         │
│ │ │ Cust A                        │       │                         │
│ │ │   ├── Order 100 (Cust A)     │       │                         │
│ │ │   ├── Order 101 (Cust A)     │       │                         │
│ │ │   └── Order 102 (Cust A)     │       │                         │
│ │ │ Cust B                        │       │                         │
│ │ │   ├── Order 200 (Cust B)     │       │                         │
│ │ │   └── Order 201 (Cust B)     │       │                         │
│ │ └──────────────────────────────┘       │                         │
│ └────────────────────────────────────────┘                         │
│ ⚡ Parent + children on SAME node = fast joins, no network hop    │
│                                                                       │
│ SQL to create interleaved tables:                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ CREATE TABLE Customers (                                      │  │
│ │   CustomerId STRING(36) NOT NULL,                            │  │
│ │   Name       STRING(100),                                     │  │
│ │   Email      STRING(255),                                     │  │
│ │ ) PRIMARY KEY (CustomerId);                                   │  │
│ │                                                               │  │
│ │ CREATE TABLE Orders (                                         │  │
│ │   CustomerId STRING(36) NOT NULL,   -- parent key first!     │  │
│ │   OrderId    STRING(36) NOT NULL,                            │  │
│ │   Amount     NUMERIC,                                         │  │
│ │   CreatedAt  TIMESTAMP,                                       │  │
│ │ ) PRIMARY KEY (CustomerId, OrderId),                         │  │
│ │   INTERLEAVE IN PARENT Customers ON DELETE CASCADE;          │  │
│ │                                                               │  │
│ │ CREATE TABLE OrderItems (                                     │  │
│ │   CustomerId STRING(36) NOT NULL,   -- grandparent key      │  │
│ │   OrderId    STRING(36) NOT NULL,   -- parent key            │  │
│ │   ItemId     STRING(36) NOT NULL,                            │  │
│ │   ProductId  STRING(36),                                      │  │
│ │   Quantity   INT64,                                           │  │
│ │ ) PRIMARY KEY (CustomerId, OrderId, ItemId),                 │  │
│ │   INTERLEAVE IN PARENT Orders ON DELETE CASCADE;             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Rules for interleaving:                                             │
│ ├── Child PK must start with parent's PK columns               │
│ ├── ON DELETE CASCADE: Delete parent → delete children         │
│ ├── ON DELETE NO ACTION: Can't delete parent if children exist │
│ ├── Max depth: 7 levels of interleaving                        │
│ └── Only interleave when you frequently JOIN parent + child    │
│                                                                       │
│ When NOT to interleave:                                             │
│ ├── Tables not frequently joined                                │
│ ├── Child table is much larger than parent                     │
│ ├── Need to query child table independently (full scans)      │
│ └── Many-to-many relationships                                 │
│                                                                       │
│ ⚡ Unique to Spanner. Cloud SQL / RDS / Aurora don't have this.   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Querying — SQL & Reads

```
┌─────────────────────────────────────────────────────────────────────┐
│           SQL DIALECTS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two SQL dialects (chosen at DATABASE creation, cannot change):     │
│                                                                       │
│ 1. GoogleSQL (default ✅):                                          │
│    ├── Spanner's native SQL dialect                              │
│    ├── Full feature support (INTERLEAVE, all types, etc.)       │
│    ├── Unique syntax differences from standard SQL              │
│    │   ├── ARRAY<INT64> instead of INT ARRAY                   │
│    │   ├── STRING(MAX) instead of VARCHAR                       │
│    │   ├── STRUCT types                                         │
│    │   └── Different function names                             │
│    └── More Spanner documentation/examples use GoogleSQL       │
│                                                                       │
│ 2. PostgreSQL-compatible:                                          │
│    ├── Familiar PostgreSQL syntax                                │
│    ├── Use existing PG tools and ORMs                           │
│    ├── Some Spanner features unavailable                       │
│    │   (e.g., INTERLEAVE syntax differs)                       │
│    └── Easier migration from Cloud SQL PostgreSQL              │
│                                                                       │
│ Standard queries:                                                   │
│ SELECT * FROM Orders                                               │
│ WHERE CustomerId = @customer_id                                   │
│ ORDER BY CreatedAt DESC                                            │
│ LIMIT 10;                                                          │
│                                                                       │
│ ⚡ Always use PARAMETERIZED queries (@param).                      │
│   Spanner caches query plans for parameterized queries.          │
│   Literal values = new plan each time = slower.                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Read Types

```
┌─────────────────────────────────────────────────────────────────────┐
│           READ TYPES                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Spanner offers different read types with different consistency    │
│ and performance trade-offs:                                       │
│                                                                       │
│ 1. STRONG READ (default):                                          │
│    ├── Sees all committed writes up to the instant of the read  │
│    ├── Global strong consistency                                │
│    ├── Highest latency (must coordinate with leader)           │
│    └── Use for: anything requiring latest data                 │
│                                                                       │
│ 2. STALE READ (bounded staleness):                                 │
│    ├── May return data up to N seconds old                     │
│    ├── Can read from nearest replica (no leader coordination)  │
│    ├── Lower latency, especially in multi-region              │
│    ├── exact_staleness: Read data at exact time T seconds ago  │
│    ├── max_staleness: Read data at most T seconds old          │
│    └── Use for: dashboards, analytics, caches, listings       │
│                                                                       │
│ 3. READ AT TIMESTAMP:                                              │
│    ├── Read data as it existed at a specific timestamp         │
│    ├── Time-travel query (within version GC window)           │
│    └── Use for: point-in-time reports, debugging              │
│                                                                       │
│ Performance impact:                                                 │
│ ├── Strong reads in multi-region: ~100ms+ (round-trip to leader)│
│ ├── Stale reads in multi-region: ~5ms (local replica)          │
│ ├── ⚡ Use stale reads with 15s staleness for read-heavy        │
│ │     workloads — huge latency reduction in multi-region!      │
│ └── Strong reads in regional: ~5-10ms (leader is close)       │
│                                                                       │
│ Code example:                                                       │
│ # Python — stale read (15 seconds staleness)                     │
│ import datetime                                                    │
│ from google.cloud import spanner                                  │
│                                                                       │
│ with database.snapshot(                                            │
│     max_staleness=datetime.timedelta(seconds=15)                 │
│ ) as snapshot:                                                     │
│     results = snapshot.execute_sql(                               │
│         "SELECT * FROM Products WHERE Category = @cat",          │
│         params={"cat": "electronics"},                           │
│         param_types={"cat": spanner.param_types.STRING},         │
│     )                                                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Transactions

```
┌─────────────────────────────────────────────────────────────────────┐
│           TRANSACTIONS                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. READ-WRITE TRANSACTIONS:                                        │
│    ├── Full ACID (Atomicity, Consistency, Isolation, Durability)│
│    ├── Serializable isolation (strongest level)                │
│    ├── Can span multiple tables and rows                       │
│    ├── Can span multiple splits and nodes                     │
│    ├── ⚡ Can span REGIONS in multi-region instances!            │
│    ├── Pessimistic locking (detects conflicts, retries)       │
│    ├── Must use client library's transaction API              │
│    └── Latency: depends on data locality and contention       │
│                                                                       │
│    # Python example                                               │
│    def transfer_funds(transaction):                               │
│        # Read                                                     │
│        row1 = transaction.read("Accounts", ["Balance"],          │
│            keyset=spanner.KeySet(keys=[["account-A"]]))          │
│        row2 = transaction.read("Accounts", ["Balance"],          │
│            keyset=spanner.KeySet(keys=[["account-B"]]))          │
│                                                                       │
│        balance_a = list(row1)[0][0]                               │
│        balance_b = list(row2)[0][0]                               │
│                                                                       │
│        # Write                                                    │
│        transaction.update("Accounts",                            │
│            columns=["AccountId", "Balance"],                     │
│            values=[                                               │
│                ["account-A", balance_a - 100],                   │
│                ["account-B", balance_b + 100],                   │
│            ])                                                     │
│                                                                       │
│    database.run_in_transaction(transfer_funds)                   │
│    # ⚡ Automatically retries on conflict!                        │
│                                                                       │
│ 2. READ-ONLY TRANSACTIONS:                                         │
│    ├── Consistent snapshot across multiple reads               │
│    ├── No locking, no conflicts                                │
│    ├── Can use strong or stale reads                           │
│    └── Use for: reports, dashboard queries, multi-table reads  │
│                                                                       │
│ 3. PARTITIONED DML (batch changes):                                │
│    ├── For large-scale UPDATE/DELETE across many rows          │
│    ├── Runs in parallel across splits                          │
│    ├── NOT ACID (each split is a separate transaction)         │
│    ├── Idempotent only (must be safe to retry per-split)      │
│    └── Use for: bulk updates, data cleanup, backfill          │
│                                                                       │
│    database.execute_partitioned_dml(                              │
│        "DELETE FROM Events WHERE CreatedAt < @cutoff",          │
│        params={"cutoff": "2025-01-01T00:00:00Z"},              │
│        param_types={"cutoff": spanner.param_types.TIMESTAMP},  │
│    )                                                              │
│                                                                       │
│ Transaction best practices:                                        │
│ ├── Keep transactions SHORT (minimize lock duration)           │
│ ├── Avoid reading large amounts of data in write transactions │
│ ├── Use read-only transactions for read-only workloads        │
│ ├── Use stale reads when possible (lower contention)          │
│ └── Design schema to minimize contention on same rows         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Secondary Indexes

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECONDARY INDEXES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Primary key = how data is physically ordered and distributed.     │
│ Secondary index = alternative lookup path for queries.            │
│                                                                       │
│ Creating indexes:                                                   │
│ CREATE INDEX OrdersByDate ON Orders (CreatedAt DESC);             │
│                                                                       │
│ CREATE INDEX OrdersByCustomerAmount                               │
│   ON Orders (CustomerId, Amount DESC);                            │
│                                                                       │
│ STORING clause (include extra columns in index):                   │
│ CREATE INDEX OrdersByDate ON Orders (CreatedAt DESC)              │
│   STORING (CustomerId, Amount, Status);                           │
│ ⚡ STORING = cover the query with index only (no base table read)│
│   Like PostgreSQL's INCLUDE clause.                              │
│                                                                       │
│ NULL_FILTERED index (skip NULLs):                                 │
│ CREATE NULL_FILTERED INDEX OrdersByPromo                          │
│   ON Orders (PromoCode);                                          │
│ ⚡ Smaller index when most rows have NULL for this column.        │
│                                                                       │
│ UNIQUE index:                                                       │
│ CREATE UNIQUE INDEX UsersByEmail ON Users (Email);               │
│                                                                       │
│ INTERLEAVED index (co-located with parent):                       │
│ CREATE INDEX OrdersByCustomer ON Orders (CustomerId, CreatedAt)  │
│   INTERLEAVE IN Customers;                                        │
│ ⚡ Index data stored next to parent rows = faster lookups        │
│   for queries filtered by CustomerId.                            │
│                                                                       │
│ Key considerations:                                                 │
│ ├── Indexes are SEPARATE tables under the hood                  │
│ ├── Every write to base table also writes to indexes            │
│ │   → More indexes = slower writes                              │
│ ├── Creating index on existing table = backfill operation       │
│ │   (runs in background, may take hours for large tables)      │
│ ├── Index creation is ONLINE (no downtime ✅)                   │
│ └── Max ~30 indexes per table recommended                       │
│                                                                       │
│ AWS/Azure: Standard B-tree indexes (same concept, different     │
│   syntax). STORING/INCLUDE and INTERLEAVE are Spanner-specific. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Backup & Restore

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKUP & RESTORE                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Backup types:                                                       │
│                                                                       │
│ 1. ON-DEMAND BACKUPS:                                               │
│    ├── Manual backup at any time                                 │
│    ├── Retained until you set an expiration or delete           │
│    ├── Max expiration: 1 year from creation                    │
│    ├── Backup is a copy — independent of source database       │
│    ├── Cannot be modified (immutable)                          │
│    └── Stored in same instance configuration (same regions)   │
│                                                                       │
│ 2. POINT-IN-TIME RECOVERY (PITR):                                  │
│    ├── Restore database to any point in time                   │
│    ├── Retention: 1 hour to 7 days (default: 1 hour)          │
│    ├── Uses version history (Spanner keeps old versions)       │
│    ├── Version GC (garbage collection) runs after retention    │
│    └── ⚡ Longer retention = more storage cost                   │
│                                                                       │
│ 3. SCHEDULED BACKUPS:                                               │
│    ├── Automated backup on a schedule                          │
│    ├── Console → Database → Backup/Restore → Backup Schedules │
│    ├── Daily, weekly, or custom CRON schedule                  │
│    ├── Set retention period per schedule                       │
│    └── ⚡ Recommended for production                             │
│                                                                       │
│ Restore:                                                            │
│ ├── Restores to a NEW database (not in-place)                  │
│ ├── Can restore to a different instance (same config)         │
│ ├── Cannot restore across instance configurations             │
│ │   (can't restore regional backup to multi-region instance)  │
│ └── Restore time depends on database size                     │
│                                                                       │
│ CLI:                                                                │
│ # Create backup                                                   │
│ gcloud spanner backups create orders-backup-20260517 \           │
│   --instance=global-orders-db \                                   │
│   --database=orders \                                             │
│   --expiration-date=2026-06-17T00:00:00Z \                      │
│   --async                                                         │
│                                                                       │
│ # List backups                                                     │
│ gcloud spanner backups list --instance=global-orders-db          │
│                                                                       │
│ # Restore backup to new database                                 │
│ gcloud spanner databases create orders-restored \                │
│   --instance=global-orders-db \                                   │
│   --source-backup=orders-backup-20260517                         │
│                                                                       │
│ # PITR — restore to specific timestamp                           │
│ gcloud spanner databases create orders-pitr \                    │
│   --instance=global-orders-db \                                   │
│   --source-database=orders \                                     │
│   --point-in-time=2026-05-17T10:30:00Z                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Security & IAM

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. IAM ROLES:                                                       │
│ ┌──────────────────────────────────┬───────────────────────────┐  │
│ │ Role                             │ Permissions               │  │
│ ├──────────────────────────────────┼───────────────────────────┤  │
│ │ roles/spanner.admin              │ Full instance + DB control│  │
│ │ roles/spanner.databaseAdmin      │ Manage DBs, schema, users │  │
│ │ roles/spanner.databaseReader     │ Read data only             │  │
│ │ roles/spanner.databaseUser       │ Read + write data          │  │
│ │ roles/spanner.viewer             │ View metadata only         │  │
│ │ roles/spanner.backupAdmin        │ Manage backups             │  │
│ │ roles/spanner.backupWriter       │ Create backups             │  │
│ └──────────────────────────────────┴───────────────────────────┘  │
│                                                                       │
│ Fine-grained access control (FGAC):                                │
│ ├── Database roles (like PostgreSQL roles)                      │
│ ├── GRANT/REVOKE on tables and columns                         │
│ ├── Row-level security via views                               │
│ └── Map IAM principals to database roles                       │
│                                                                       │
│ CREATE ROLE order_reader;                                          │
│ GRANT SELECT ON TABLE Orders TO ROLE order_reader;                │
│ GRANT SELECT(Name, Email) ON TABLE Customers TO ROLE order_reader;│
│ -- Column-level access: can only read Name and Email             │
│                                                                       │
│ 2. ENCRYPTION:                                                      │
│    ├── All data encrypted at rest (Google-managed default)      │
│    ├── CMEK available (Cloud KMS)                               │
│    ├── Data encrypted in transit (TLS between client/Spanner)  │
│    └── Data encrypted between Spanner replicas                 │
│                                                                       │
│ 3. VPC SERVICE CONTROLS:                                           │
│    ├── Create perimeter around Spanner resources               │
│    ├── Prevent data exfiltration                                │
│    └── Restrict which projects/networks can access Spanner     │
│                                                                       │
│ 4. AUDIT LOGGING:                                                   │
│    ├── Admin Activity logs (schema changes, config changes)    │
│    │   Always enabled, free                                     │
│    ├── Data Access logs (reads and writes to data)             │
│    │   Must be enabled, costs apply                             │
│    └── Viewable in Cloud Logging                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Monitoring & Performance

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Spanner → Instance → System Insights:                   │
│                                                                       │
│ Key metrics:                                                        │
│ ├── CPU utilization (%)                                          │
│ │   ⚡ Keep below 65% (high priority) for optimal performance    │
│ │   Spanner recommends: Scale up if CPU > 65%                   │
│ │   (Remaining 35% is for background tasks like compaction)    │
│ │                                                                 │
│ ├── Storage utilization (bytes)                                  │
│ │   ⚡ Max 4 TB per node — scale before hitting limit            │
│ │                                                                 │
│ ├── Read/Write latency (ms)                                     │
│ │   ├── P50, P99, P99.9 latencies                              │
│ │   └── Sudden spike = possible hotspot or contention          │
│ │                                                                 │
│ ├── Operations/second (reads, writes)                           │
│ ├── API request count and errors                                │
│ └── Transaction commit latency                                  │
│                                                                       │
│ Query Statistics:                                                   │
│ Console → Database → Query Statistics:                            │
│ ├── Top queries by CPU usage                                    │
│ ├── Query execution count and latency                          │
│ ├── Scanned rows vs returned rows (efficiency)                 │
│ ├── Full scan detection ⚠️                                       │
│ └── ⚡ If scanned >> returned: Add index or refine query         │
│                                                                       │
│ Transaction Statistics:                                             │
│ ├── Commit latency distribution                                 │
│ ├── Lock wait time                                               │
│ ├── Aborted transactions (contention!)                          │
│ └── ⚡ High abort rate = hot rows or long-held locks             │
│                                                                       │
│ Hotspot detection:                                                  │
│ ├── Key Visualizer: Console → Database → Key Visualizer       │
│ │   Shows heatmap of read/write activity across key range      │
│ │   ⚡ Bright horizontal band = hotspot!                         │
│ ├── Causes: Monotonic keys, popular rows, sequential inserts  │
│ └── Fix: Change primary key design, add caching layer         │
│                                                                       │
│ Alerts:                                                             │
│ ├── CPU > 65%: Scale up (add PU/nodes)                         │
│ ├── Storage > 70%: Scale up or increase PITR retention window │
│ ├── Latency P99 > threshold: Investigate queries              │
│ └── Transaction abort rate > threshold: Investigate contention│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Terraform & CLI

### Terraform

```hcl
# === Spanner Instance (Regional) ===
resource "google_spanner_instance" "main" {
  name             = "orders-instance"
  config           = "regional-asia-south1"
  display_name     = "Orders Instance"
  processing_units = 1000   # 1 node

  # Or use autoscaling:
  autoscaling_config {
    autoscaling_limits {
      min_processing_units = 1000
      max_processing_units = 5000
    }
    autoscaling_targets {
      high_priority_cpu_utilization_percent = 65
      storage_utilization_percent           = 70
    }
  }

  labels = {
    environment = "prod"
    team        = "platform"
  }
}

# === Spanner Instance (Multi-Region) ===
resource "google_spanner_instance" "global" {
  name             = "global-orders"
  config           = "nam-eur-asia1"
  display_name     = "Global Orders"
  processing_units = 3000   # 3 nodes

  labels = {
    environment = "prod"
  }
}

# === Database (GoogleSQL) ===
resource "google_spanner_database" "orders" {
  instance = google_spanner_instance.main.name
  name     = "orders"

  version_retention_period = "7d"   # PITR window
  deletion_protection      = true

  ddl = [
    <<-SQL
    CREATE TABLE Customers (
      CustomerId STRING(36) NOT NULL,
      Name       STRING(100),
      Email      STRING(255) NOT NULL,
      CreatedAt  TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp = true),
    ) PRIMARY KEY (CustomerId)
    SQL
    ,
    <<-SQL
    CREATE TABLE Orders (
      CustomerId STRING(36) NOT NULL,
      OrderId    STRING(36) NOT NULL,
      Amount     NUMERIC,
      Status     STRING(20),
      CreatedAt  TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp = true),
    ) PRIMARY KEY (CustomerId, OrderId),
    INTERLEAVE IN PARENT Customers ON DELETE CASCADE
    SQL
    ,
    <<-SQL
    CREATE TABLE OrderItems (
      CustomerId STRING(36) NOT NULL,
      OrderId    STRING(36) NOT NULL,
      ItemId     STRING(36) NOT NULL,
      ProductId  STRING(36),
      Quantity   INT64,
      Price      NUMERIC,
    ) PRIMARY KEY (CustomerId, OrderId, ItemId),
    INTERLEAVE IN PARENT Orders ON DELETE CASCADE
    SQL
    ,
    "CREATE INDEX OrdersByStatus ON Orders (Status)",
    "CREATE INDEX OrdersByDate ON Orders (CreatedAt DESC) STORING (CustomerId, Amount, Status)",
    "CREATE UNIQUE INDEX CustomersByEmail ON Customers (Email)",
  ]
}

# === Database (PostgreSQL dialect) ===
resource "google_spanner_database" "pg_db" {
  instance         = google_spanner_instance.main.name
  name             = "analytics"
  database_dialect = "POSTGRESQL"

  deletion_protection = false
}

# === IAM ===
resource "google_spanner_database_iam_member" "reader" {
  instance = google_spanner_instance.main.name
  database = google_spanner_database.orders.name
  role     = "roles/spanner.databaseReader"
  member   = "serviceAccount:analytics-sa@my-project.iam.gserviceaccount.com"
}

resource "google_spanner_database_iam_member" "writer" {
  instance = google_spanner_instance.main.name
  database = google_spanner_database.orders.name
  role     = "roles/spanner.databaseUser"
  member   = "serviceAccount:orders-api@my-project.iam.gserviceaccount.com"
}

# === Backup Schedule ===
resource "google_spanner_backup_schedule" "daily" {
  instance = google_spanner_instance.main.name
  database = google_spanner_database.orders.name
  name     = "daily-backup"

  retention_duration = "2592000s"   # 30 days

  spec {
    cron_spec {
      text = "0 2 * * *"   # Daily at 2:00 AM UTC
    }
  }

  full_backup_spec {}
}
```

### gcloud CLI Reference

```bash
# =============================================
# INSTANCE MANAGEMENT
# =============================================

# Create regional instance
gcloud spanner instances create orders-instance \
  --config=regional-asia-south1 \
  --description="Orders Instance" \
  --processing-units=1000

# Create multi-region instance
gcloud spanner instances create global-orders \
  --config=nam-eur-asia1 \
  --description="Global Orders" \
  --nodes=3

# List instances
gcloud spanner instances list

# Describe instance
gcloud spanner instances describe orders-instance

# Scale up (zero downtime)
gcloud spanner instances update orders-instance \
  --processing-units=3000

# Scale down
gcloud spanner instances update orders-instance \
  --processing-units=1000

# Delete instance (⚠️ deletes all databases!)
gcloud spanner instances delete orders-instance

# =============================================
# DATABASE MANAGEMENT
# =============================================

# Create database
gcloud spanner databases create orders \
  --instance=orders-instance

# Create database with initial schema
gcloud spanner databases create orders \
  --instance=orders-instance \
  --ddl="CREATE TABLE Customers (CustomerId STRING(36) NOT NULL, Name STRING(100)) PRIMARY KEY (CustomerId)"

# Update schema (add column)
gcloud spanner databases ddl update orders \
  --instance=orders-instance \
  --ddl="ALTER TABLE Customers ADD COLUMN Phone STRING(20)"

# Create index
gcloud spanner databases ddl update orders \
  --instance=orders-instance \
  --ddl="CREATE INDEX CustomersByEmail ON Customers (Email)"

# List databases
gcloud spanner databases list --instance=orders-instance

# Describe database
gcloud spanner databases describe orders --instance=orders-instance

# Delete database
gcloud spanner databases delete orders --instance=orders-instance

# =============================================
# QUERYING
# =============================================

# Execute SQL query
gcloud spanner databases execute-sql orders \
  --instance=orders-instance \
  --sql="SELECT * FROM Customers LIMIT 10"

# =============================================
# BACKUPS
# =============================================

# Create backup
gcloud spanner backups create orders-backup-20260517 \
  --instance=orders-instance \
  --database=orders \
  --expiration-date=2026-06-17T00:00:00Z \
  --async

# List backups
gcloud spanner backups list --instance=orders-instance

# Restore backup to new database
gcloud spanner databases create orders-restored \
  --instance=orders-instance \
  --source-backup=orders-backup-20260517

# PITR — restore to specific point in time
gcloud spanner databases create orders-pitr \
  --instance=orders-instance \
  --source-database=orders \
  --point-in-time=2026-05-17T10:30:00Z

# Delete backup
gcloud spanner backups delete orders-backup-20260517 \
  --instance=orders-instance

# =============================================
# IAM
# =============================================

# Grant read access
gcloud spanner databases add-iam-policy-binding orders \
  --instance=orders-instance \
  --member=serviceAccount:sa@project.iam.gserviceaccount.com \
  --role=roles/spanner.databaseReader

# =============================================
# OPERATIONS
# =============================================

# List long-running operations (schema changes, backups)
gcloud spanner operations list --instance=orders-instance
```

---

## Part 14: Real-World Patterns

### Startup

```
┌─────────────────────────────────────────────────────────────────────┐
│ STARTUP — SHOULD YOU USE SPANNER?                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Almost certainly NO.                                                │
│                                                                       │
│ ├── Use Cloud SQL PostgreSQL instead ($100-300/month)           │
│ ├── Spanner minimum: ~$65/month (100 PU) but limited capacity  │
│ │   Realistic minimum for production: ~$650/month (1 node)     │
│ ├── Cloud SQL handles millions of rows easily                  │
│ ├── You don't need 99.999% availability yet                    │
│ ├── You don't need multi-region yet                             │
│ └── Switch to Spanner when you ACTUALLY hit Cloud SQL limits  │
│                                                                       │
│ Only use Spanner at startup if:                                    │
│ ├── Global users from day 1 (gaming, fintech)                  │
│ ├── Need strong consistency across regions                     │
│ └── Funded and can afford $1000+/month for database            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Mid-Size

```
┌─────────────────────────────────────────────────────────────────────┐
│ MID-SIZE (50-100 developers) — EVALUATE SPANNER                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Consider Spanner when:                                              │
│ ├── Cloud SQL hitting vertical scaling limits (96 vCPUs)       │
│ ├── Need multi-region for DR or user proximity                 │
│ ├── Transaction throughput exceeds single-node capacity        │
│ └── Data exceeding 64 TB                                       │
│                                                                       │
│ Instance:                                                            │
│ ├── Regional (asia-south1), 1-3 nodes                           │
│ ├── Autoscaler enabled (1-5 nodes)                              │
│ ├── GoogleSQL dialect                                            │
│ └── ~$650-2,000/month                                           │
│                                                                       │
│ Schema:                                                              │
│ ├── UUID primary keys (GENERATE_UUID())                         │
│ ├── Interleaved tables for parent-child (Customers → Orders)  │
│ ├── Covering indexes with STORING clause                       │
│ └── Stale reads (15s) for dashboard/listing queries           │
│                                                                       │
│ Backups:                                                             │
│ ├── Daily automated backups (30-day retention)                 │
│ ├── PITR enabled (7-day version retention)                     │
│ └── On-demand backup before schema changes                    │
│                                                                       │
│ Monitoring:                                                         │
│ ├── CPU utilization alerts at 65%                               │
│ ├── Key Visualizer for hotspot detection                       │
│ └── Query Statistics for slow query optimization               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Enterprise

```
┌─────────────────────────────────────────────────────────────────────┐
│ ENTERPRISE (500+ developers) — SPANNER SWEET SPOT                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance:                                                            │
│ ├── Multi-region (nam-eur-asia1 or custom dual-region)         │
│ ├── 5-20+ nodes with autoscaler                                │
│ ├── 99.999% SLA                                                 │
│ ├── CMEK encryption (Cloud KMS)                                │
│ └── VPC Service Controls perimeter                              │
│                                                                       │
│ Architecture:                                                        │
│ ├── Separate instances per domain (orders, inventory, users)   │
│ ├── Interleaved tables for performance-critical paths          │
│ ├── Stale reads for read-heavy workloads                       │
│ ├── Change Streams for event-driven architecture               │
│ │   (Spanner → Pub/Sub → downstream services)                 │
│ └── Data Boost for analytics (isolated compute for queries)   │
│                                                                       │
│ Schema governance:                                                  │
│ ├── Schema changes via CI/CD (Terraform or Liquibase)         │
│ ├── Schema review process (PR-based DDL changes)              │
│ ├── Fine-grained access control (database roles)              │
│ └── Column-level permissions for sensitive data               │
│                                                                       │
│ Backups & DR:                                                        │
│ ├── Automated daily backups (90-day retention)                 │
│ ├── PITR with 7-day version retention                          │
│ ├── Multi-region instance = built-in DR                        │
│ ├── Quarterly restore drills                                    │
│ └── RPO ≈ 0 (synchronous replication), RTO < 1 min            │
│                                                                       │
│ Monitoring:                                                         │
│ ├── CPU alerts at 55% (proactive scaling)                      │
│ ├── Key Visualizer daily review                                │
│ ├── Transaction abort rate monitoring                          │
│ ├── Custom Grafana dashboards with Spanner metrics            │
│ └── PagerDuty for on-call alerts                               │
│                                                                       │
│ Cost optimization:                                                  │
│ ├── 3-year CUD (40% discount)                                  │
│ ├── Autoscaler for off-peak scaling                            │
│ ├── Stale reads to reduce leader load                         │
│ └── Data Boost for analytics (don't use main capacity)       │
│                                                                       │
│ Monthly cost: $5,000-50,000+ depending on scale                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌──────────────────────────────────────────────────────────────────────┐
│ CLOUD SPANNER — QUICK REFERENCE                                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ What: Globally distributed, strongly consistent relational database  │
│ SQL: GoogleSQL (native) or PostgreSQL-compatible dialect            │
│ Scale: Horizontal (add nodes), unlimited data                       │
│ SLA: 99.99% (regional) | 99.999% (multi-region)                    │
│                                                                        │
│ Capacity:                                                             │
│ ├── Processing Unit (PU): Smallest compute unit                      │
│ ├── 1 Node = 1000 PU                                                 │
│ ├── Min: 100 PU | Scale: 100 PU increments (< 1000), 1000 (≥ 1000)│
│ ├── Storage: 4 TB per node (regional)                                │
│ └── Autoscaler available ✅                                           │
│                                                                        │
│ Architecture: Data → Splits → Nodes → Paxos replication             │
│ TrueTime: GPS + atomic clocks → globally ordered transactions       │
│                                                                        │
│ Schema design:                                                        │
│ ├── ⚠️ NO monotonic keys (use UUID, hash-prefix, bit-reverse)       │
│ ├── INTERLEAVE tables for parent-child co-location                   │
│ ├── Schema changes are ONLINE (zero downtime)                        │
│ └── STORING clause on indexes for covering queries                   │
│                                                                        │
│ Reads: Strong (default) | Stale (lower latency) | At timestamp     │
│ Transactions: Read-write (ACID) | Read-only | Partitioned DML      │
│ Backups: On-demand | Scheduled | PITR (1h-7d version retention)    │
│                                                                        │
│ Key limitations:                                                      │
│ ├── EXPENSIVE ($650+/month minimum for production)                   │
│ ├── No full MySQL/PG compatibility (limited extensions)              │
│ ├── Cannot change instance config after creation                     │
│ ├── Cannot change database dialect after creation                    │
│ ├── Write latency: 5-10ms (regional), 100-200ms (multi-region)      │
│ └── Learning curve for schema design (interleaving, keys)            │
│                                                                        │
│ AWS ↔ GCP:                                                             │
│ ├── No direct equivalent (Aurora is closest but single-region)       │
│ ├── CockroachDB = open-source Spanner-like (no TrueTime)            │
│ └── DynamoDB Global Tables ≈ eventual consistency (not strong)       │
│                                                                        │
│ Azure ↔ GCP:                                                           │
│ ├── Cosmos DB = multi-model, offers strong consistency mode          │
│ └── Cosmos is NoSQL-first; Spanner is relational-first              │
│                                                                        │
│ When to use:                                                          │
│ ├── Need horizontal scaling beyond Cloud SQL                         │
│ ├── Need multi-region strong consistency                              │
│ ├── Need 99.999% availability                                        │
│ ├── Global financial, gaming, inventory systems                      │
│ └── Can afford $1000+/month for database                             │
│                                                                        │
│ When NOT to use:                                                      │
│ ├── Cloud SQL handles your workload → use Cloud SQL                  │
│ ├── Budget < $1000/month → use Cloud SQL                             │
│ ├── Need full MySQL/PG extension support → use Cloud SQL             │
│ └── Simple key-value lookups → use Firestore or Bigtable             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Spanner Resources

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING SPANNER FROM THE CONSOLE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ SCALING PROCESSING UNITS:                                           │
│ Console → Spanner → [your instance] → Instance Overview            │
│ 1. Click "Edit instance" (pencil icon or button at top)           │
│ 2. Under "Configure compute capacity":                             │
│    ├── Change Processing Units (e.g., 1000 → 3000)               │
│    ├── Or change Nodes (e.g., 1 → 3)                              │
│    └── If autoscaling is on, adjust Min/Max nodes instead         │
│ 3. Click "Save"                                                    │
│ ⚡ Scaling is ZERO DOWNTIME — takes effect within minutes.         │
│ ⚡ You can scale up or down at any time.                            │
│                                                                       │
│ DELETING A DATABASE:                                                │
│ Console → Spanner → [your instance] → Databases tab               │
│ 1. Click on the database name (e.g., "orders")                    │
│ 2. Click "Delete database" (trash icon or dropdown menu)          │
│ 3. Type the database name to confirm                               │
│ 4. Click "Delete"                                                   │
│ ⚠️ This permanently deletes ALL data in the database!              │
│ ⚠️ Delete any backups separately if you want to clean those up.    │
│                                                                       │
│ DELETING A SPANNER INSTANCE:                                       │
│ Console → Spanner → [your instance] → Instance Overview            │
│ 1. You MUST delete ALL databases in the instance first            │
│    ├── Instance cannot be deleted while databases exist            │
│    └── Delete each database one by one (see steps above)          │
│ 2. Once all databases are gone, click "Delete instance"           │
│ 3. Type the instance name to confirm                               │
│ 4. Click "Delete"                                                   │
│ ⚠️ This is irreversible — the instance and all its data are gone. │
│                                                                       │
│ MANAGING BACKUPS:                                                   │
│ Console → Spanner → [your instance] → [your database] → Backup   │
│ 1. View existing backups and their expiration dates               │
│ 2. To create a manual backup:                                      │
│    ├── Click "Create backup"                                      │
│    ├── Set a backup name and expiration date                      │
│    └── Click "Create"                                              │
│ 3. To restore a backup:                                            │
│    ├── Click on the backup name                                   │
│    ├── Click "Restore"                                             │
│    ├── Choose a new database name (cannot overwrite existing)     │
│    └── Click "Restore"                                             │
│ 4. To delete a backup:                                             │
│    ├── Select the backup → click "Delete"                         │
│    └── ⚡ Backups count against storage costs until deleted        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Spanner Emulator for Local Development

```
┌─────────────────────────────────────────────────────────────────────┐
│           SPANNER EMULATOR                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The Spanner emulator lets you develop and test LOCALLY without     │
│ incurring any costs. Since Spanner is expensive ($650+/month for   │
│ a single node), the emulator is essential for development.         │
│                                                                       │
│ Why use the emulator?                                               │
│ ├── FREE — no billing for local development                       │
│ ├── Fast iteration — no network latency to cloud                  │
│ ├── Offline development — works without internet                  │
│ ├── CI/CD testing — run integration tests in pipelines            │
│ └── Safe — experiment without touching production data            │
│                                                                       │
│ ⚠️ Limitations:                                                     │
│ ├── No TrueTime (timestamps use local clock)                      │
│ ├── No multi-region replication                                    │
│ ├── Single-node only (no split testing)                           │
│ ├── No IAM or security features                                   │
│ └── Not for performance benchmarking                               │
│                                                                       │
│ Setup:                                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Start the emulator                                          │   │
│ │ gcloud emulators spanner start                                │   │
│ │                                                               │   │
│ │ # Set environment variables to point your app to the emulator │   │
│ │ gcloud emulators spanner env-init                             │   │
│ │                                                               │   │
│ │ # Typical output — set these in your shell:                   │   │
│ │ export SPANNER_EMULATOR_HOST=localhost:9010                   │   │
│ │                                                               │   │
│ │ # Create an instance on the emulator                          │   │
│ │ gcloud spanner instances create test-instance \               │   │
│ │   --config=emulator-config \                                  │   │
│ │   --description="Test" \                                      │   │
│ │   --nodes=1                                                   │   │
│ │                                                               │   │
│ │ # Create a database on the emulator                           │   │
│ │ gcloud spanner databases create test-db \                     │   │
│ │   --instance=test-instance                                    │   │
│ │                                                               │   │
│ │ # Your app code connects to the emulator automatically        │   │
│ │ # when SPANNER_EMULATOR_HOST is set.                          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Always use the emulator for development and testing.             │
│   Only connect to real Spanner for staging/production.             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Additional gcloud Commands

```bash
# =============================================
# SCALING (update processing units or nodes)
# =============================================

# Scale up processing units (zero downtime)
gcloud spanner instances update orders-instance \
  --processing-units=3000

# Scale down processing units
gcloud spanner instances update orders-instance \
  --processing-units=1000

# Scale using nodes instead
gcloud spanner instances update orders-instance \
  --nodes=5

# Update instance description
gcloud spanner instances update orders-instance \
  --description="Updated Orders Instance"

# =============================================
# DELETING DATABASES
# =============================================

# Delete a specific database
gcloud spanner databases delete orders \
  --instance=orders-instance

# ⚠️ This permanently deletes ALL data in the database.
# ⚠️ There is NO undo. Make sure you have backups!

# Delete database without confirmation prompt (for scripts)
gcloud spanner databases delete orders \
  --instance=orders-instance \
  --quiet

# =============================================
# DELETING INSTANCES
# =============================================

# Delete an instance (all databases must be deleted first!)
gcloud spanner instances delete orders-instance

# ⚠️ If databases still exist, you'll get an error:
#    "Instance is not empty. Delete all databases first."

# Step-by-step cleanup:
# 1. List databases in the instance
gcloud spanner databases list --instance=orders-instance

# 2. Delete each database
gcloud spanner databases delete orders --instance=orders-instance
gcloud spanner databases delete analytics --instance=orders-instance

# 3. Now delete the instance
gcloud spanner instances delete orders-instance

# Delete without confirmation prompt (for scripts)
gcloud spanner instances delete orders-instance --quiet
```

---

## What's Next?

Continue to **Chapter 26: Firestore & Datastore** → `26-firestore.md`
