# Chapter 29: AlloyDB

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AlloyDB Fundamentals](#part-1-alloydb-fundamentals)
- [Part 2: Architecture — How AlloyDB Works](#part-2-architecture--how-alloydb-works)
- [Part 3: Clusters & Instances](#part-3-clusters--instances)
- [Part 4: Creating an AlloyDB Cluster](#part-4-creating-an-alloydb-cluster)
- [Part 5: Instance Types — Primary & Read Pool](#part-5-instance-types--primary--read-pool)
- [Part 6: Connecting to AlloyDB](#part-6-connecting-to-alloydb)
- [Part 7: Columnar Engine](#part-7-columnar-engine)
- [Part 8: High Availability & Failover](#part-8-high-availability--failover)
- [Part 9: Cross-Region Replication](#part-9-cross-region-replication)
- [Part 10: Backups & Restore](#part-10-backups--restore)
- [Part 11: Maintenance & Updates](#part-11-maintenance--updates)
- [Part 12: Security & IAM](#part-12-security--iam)
- [Part 13: Monitoring & Performance](#part-13-monitoring--performance)
- [Part 14: Terraform & CLI](#part-14-terraform--cli)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

AlloyDB for PostgreSQL is Google Cloud's fully managed, PostgreSQL-compatible relational database designed for demanding enterprise workloads. It combines the familiarity of PostgreSQL with Google's infrastructure to deliver up to 4x faster transactional performance and 100x faster analytical queries compared to standard PostgreSQL — while remaining 100% PostgreSQL-compatible. Think of it as "Cloud SQL for PostgreSQL on steroids" with a re-engineered storage layer, integrated columnar engine, and AI/ML capabilities built in.

```
┌─────────────────────────────────────────────────────────────────────┐
│ WHAT YOU'LL LEARN                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ├── 1. What is AlloyDB and when to use it vs Cloud SQL / Spanner  │
│ ├── 2. Architecture: disaggregated compute + storage               │
│ ├── 3. Clusters and instances                                      │
│ ├── 4. Creating a cluster (console walkthrough)                    │
│ ├── 5. Instance types: primary and read pool instances             │
│ ├── 6. Connecting from applications                                │
│ ├── 7. Columnar engine for analytics                               │
│ ├── 8. High availability and automatic failover                    │
│ ├── 9. Cross-region replication for disaster recovery              │
│ ├── 10. Backups, PITR, and restore                                 │
│ ├── 11. Maintenance and version management                        │
│ ├── 12. Security, IAM, encryption, and networking                  │
│ ├── 13. Monitoring and performance tuning                          │
│ ├── 14. Terraform and gcloud CLI                                   │
│ └── 15. Real-world architecture patterns                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: AlloyDB Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS ALLOYDB?                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ AlloyDB = PostgreSQL compatibility + Google-engineered storage     │
│           + columnar engine + ML integration                       │
│                                                                       │
│ Key characteristics:                                                │
│ ├── 100% PostgreSQL compatible (v14, v15)                          │
│ │   → Use any PostgreSQL driver, tool, extension, ORM             │
│ │   → pg_dump, pgAdmin, psql, Prisma, SQLAlchemy all work        │
│ ├── Up to 4x faster transactions than standard PostgreSQL          │
│ ├── Up to 100x faster analytical queries (columnar engine)         │
│ ├── Fully managed — no OS patching, no storage provisioning       │
│ ├── 99.99% availability SLA (with HA enabled)                     │
│ ├── Automatic storage scaling (no pre-provisioning)                │
│ ├── Built-in ML functions (vertex_ai_integration extension)        │
│ ├── Cross-region replication for DR                                │
│ ├── PITR up to 14 days                                             │
│ └── VPC-native (private IP only, like Memorystore)                 │
│                                                                       │
│ When to use AlloyDB:                                                │
│ ├── Enterprise PostgreSQL workloads needing high performance       │
│ ├── Mixed OLTP + OLAP workloads (transactional + analytical)       │
│ ├── Migrating from Oracle/SQL Server → want PostgreSQL perf      │
│ ├── Applications needing both fast writes AND complex analytics    │
│ ├── ML/AI integration with database (embedding, predictions)       │
│ └── Need 99.99% SLA with automatic failover                        │
│                                                                       │
│ When NOT to use AlloyDB:                                            │
│ ├── Small/simple workloads → Cloud SQL (cheaper)                   │
│ ├── Need MySQL or SQL Server → Cloud SQL                           │
│ ├── Need global distribution → Cloud Spanner                       │
│ ├── Need NoSQL/document model → Firestore                          │
│ ├── Budget-constrained → Cloud SQL (AlloyDB has higher min cost)  │
│ └── Need multi-region writes → Cloud Spanner                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### AlloyDB vs Cloud SQL vs Cloud Spanner

```
┌─────────────────────────────────────────────────────────────────────┐
│           GCP RELATIONAL DATABASE COMPARISON                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────┬──────────────┬──────────────┬────────────┐│
│ │ Feature              │ Cloud SQL    │ AlloyDB      │ Spanner    ││
│ ├──────────────────────┼──────────────┼──────────────┼────────────┤│
│ │ Compatibility        │ MySQL/PG/    │ PostgreSQL   │ GoogleSQL/ ││
│ │                      │ SQL Server   │ only         │ PG-compat  ││
│ │ Performance          │ Standard PG  │ 4x PG OLTP   │ Unlimited  ││
│ │                      │              │ 100x PG OLAP │ horizontal ││
│ │ Storage engine       │ Standard InnoDB│ Custom      │ Custom     ││
│ │                      │ / PG storage │ (log-based)  │ (Colossus) ││
│ │ Max storage          │ 64 TB        │ 128 TB       │ Unlimited  ││
│ │ Columnar engine      │ ❌ No         │ ✅ Built-in   │ ❌ No       ││
│ │ ML integration       │ ❌ Basic      │ ✅ Vertex AI  │ ❌ No       ││
│ │ HA SLA               │ 99.95%       │ 99.99%       │ 99.999%    ││
│ │ Multi-region writes  │ ❌ No         │ ❌ No         │ ✅ Yes      ││
│ │ Cross-region read     │ ✅ Replicas   │ ✅ Replicas   │ ✅ Built-in ││
│ │ Auto-storage scaling │ ✅ Yes        │ ✅ Yes        │ ✅ Yes      ││
│ │ Serverless option    │ ❌ No         │ ❌ No         │ ❌ No       ││
│ │ Connection method    │ Public/Private│ Private only │ Public/    ││
│ │                      │ IP, Auth Proxy│ Auth Proxy   │ Private    ││
│ │ Min cost (approx)    │ ~$7/month    │ ~$200/month  │ ~$65/month ││
│ │ Best for             │ Small-medium │ Enterprise   │ Global     ││
│ │                      │ workloads    │ mixed OLTP/  │ scale,     ││
│ │                      │              │ OLAP         │ strong     ││
│ │                      │              │              │ consistency││
│ └──────────────────────┴──────────────┴──────────────┴────────────┘│
│                                                                       │
│ Decision tree:                                                      │
│ ├── Budget-sensitive or simple app? → Cloud SQL                   │
│ ├── Need MySQL or SQL Server? → Cloud SQL                         │
│ ├── Enterprise PG with analytics? → AlloyDB ✅                    │
│ ├── Global scale with strong consistency? → Spanner               │
│ └── Not sure between AlloyDB and Cloud SQL PG?                    │
│     → Start with Cloud SQL, migrate to AlloyDB if you need        │
│       performance (100% PG-compatible = easy migration)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

```
┌──────────────────────┬────────────────┬──────────────────┬──────────────┐
│ Feature              │ GCP AlloyDB    │ AWS Aurora PG    │ Azure Cosmos  │
│                      │                │                  │ DB for PG     │
├──────────────────────┼────────────────┼──────────────────┼──────────────┤
│ PG compatibility     │ ✅ 100%         │ ✅ Wire-compat    │ ✅ Wire-compat│
│ Columnar engine      │ ✅ Built-in     │ ❌ No             │ ❌ No         │
│ ML integration       │ ✅ Vertex AI    │ ✅ SageMaker      │ ❌ Limited    │
│ Storage auto-scaling │ ✅ Yes          │ ✅ Yes            │ ✅ Yes        │
│ HA SLA               │ 99.99%         │ 99.99%           │ 99.99%       │
│ Read replicas        │ ✅ Read pools   │ ✅ Up to 15       │ ✅ Read regions│
│ Cross-region         │ ✅ Secondary    │ ✅ Global DB      │ ✅ Multi-region│
│ Serverless           │ ❌ No           │ ✅ Aurora Serverless│ ✅ Serverless │
│ Max storage          │ 128 TB         │ 128 TB           │ Unlimited    │
│ PITR retention       │ 14 days        │ 35 days          │ 30 days      │
└──────────────────────┴────────────────┴──────────────────┴──────────────┘
```

### Pricing Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALLOYDB PRICING                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌───────────────────────┬────────────────────────────────────────┐ │
│ │ Component             │ Price (approximate, us-central1)       │ │
│ ├───────────────────────┼────────────────────────────────────────┤ │
│ │ Primary instance      │ Based on vCPU + memory                │ │
│ │ (2 vCPU, 16 GB RAM)   │ ~$0.30/hr (~$216/month)               │ │
│ │ (4 vCPU, 32 GB RAM)   │ ~$0.60/hr (~$432/month)               │ │
│ │ (16 vCPU, 128 GB RAM) │ ~$2.40/hr (~$1,728/month)             │ │
│ │                       │                                        │ │
│ │ Read pool instance    │ Same pricing as primary (per instance) │ │
│ │                       │                                        │ │
│ │ Storage               │ $0.00034/GB/hr (~$0.245/GB/month)     │ │
│ │                       │ Auto-scaling, pay for used only        │ │
│ │                       │                                        │ │
│ │ Networking            │ Standard GCP egress rates              │ │
│ │                       │                                        │ │
│ │ Backup storage        │ $0.00008/GB/hr (~$0.058/GB/month)     │ │
│ │                       │ First backup = free (included)         │ │
│ └───────────────────────┴────────────────────────────────────────┘ │
│                                                                       │
│ Minimum production setup:                                           │
│ ├── Primary (2 vCPU): ~$216/month                                 │
│ ├── HA standby (auto): ~$216/month (HA doubles compute cost)      │
│ ├── Storage (50 GB): ~$12/month                                    │
│ └── Total: ~$444/month (minimum HA cluster)                        │
│                                                                       │
│ ⚡ More expensive than Cloud SQL but significantly faster.          │
│ ⚡ Storage auto-scales — you don't pre-provision.                  │
│ ⚡ No free tier.                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture — How AlloyDB Works

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALLOYDB ARCHITECTURE                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ AlloyDB separates compute and storage (like Aurora):               │
│                                                                       │
│                    ┌───────────────────────────────┐                │
│                    │     COMPUTE LAYER              │                │
│                    │                               │                │
│  ┌──────────────┐  │  ┌──────────────┐            │                │
│  │   Primary    │  │  │   Standby    │            │                │
│  │   Instance   │  │  │   Instance   │            │                │
│  │ (read/write) │  │  │ (HA failover)│            │                │
│  │              │  │  │              │            │                │
│  │ ┌──────────┐ │  │  │ ┌──────────┐ │            │                │
│  │ │ PG Engine│ │  │  │ │ PG Engine│ │            │                │
│  │ │ + Ultra  │ │  │  │ │ + Ultra  │ │            │                │
│  │ │   Cache  │ │  │  │ │   Cache  │ │            │                │
│  │ └──────────┘ │  │  │ └──────────┘ │            │                │
│  └──────┬───────┘  │  └──────┬───────┘            │                │
│         │          │         │                     │                │
│  ┌──────┴──────────┴─────────┴──────────────────┐ │                │
│  │  Read Pool Instances                          │ │                │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐     │ │                │
│  │  │ Read     │ │ Read     │ │ Read     │     │ │                │
│  │  │ Pool 1   │ │ Pool 2   │ │ Pool N   │     │ │                │
│  │  │ + Ultra  │ │ + Ultra  │ │ + Ultra  │     │ │                │
│  │  │   Cache  │ │   Cache  │ │   Cache  │     │ │                │
│  │  └──────────┘ └──────────┘ └──────────┘     │ │                │
│  └──────────────────┬───────────────────────────┘ │                │
│                     │                              │                │
│                     │ WAL (write-ahead log)        │                │
│                     ▼                              │                │
│  ┌──────────────────────────────────────────────┐ │                │
│  │          STORAGE LAYER                        │ │                │
│  │  ┌────────────────────────────────────────┐  │ │                │
│  │  │  Google Distributed Storage             │  │ │                │
│  │  │  (log-structured, disaggregated)        │  │ │                │
│  │  │                                         │  │ │                │
│  │  │  ├── WAL logs replicated 3x             │  │ │                │
│  │  │  ├── Automatic compaction               │  │ │                │
│  │  │  ├── Auto-scaling (no provisioning)     │  │ │                │
│  │  │  └── Up to 128 TB                       │  │ │                │
│  │  └────────────────────────────────────────┘  │ │                │
│  └──────────────────────────────────────────────┘ │                │
│                    └───────────────────────────────┘                │
│                                                                       │
│ Key architectural innovations:                                      │
│                                                                       │
│ 1. LOG-BASED STORAGE                                               │
│    ├── Only WAL logs are written to storage (not full pages)       │
│    ├── Storage layer processes WAL to reconstruct data             │
│    ├── Reduces write amplification by 4x vs standard PostgreSQL   │
│    └── Storage handles compaction/garbage collection               │
│                                                                       │
│ 2. ULTRA-FAST CACHE                                                │
│    ├── Multi-tier cache on each compute instance                   │
│    ├── Tier 1: RAM (buffer pool, shared_buffers equivalent)        │
│    ├── Tier 2: Local SSD (ultra-fast cache, survives restarts)     │
│    ├── Cache warms automatically from storage layer                │
│    └── Read pool instances have their OWN cache                    │
│                                                                       │
│ 3. COMPUTE-STORAGE SEPARATION                                      │
│    ├── Add read pool instances without copying data               │
│    ├── Scale compute independently from storage                    │
│    ├── Failover is fast (standby already has cache)               │
│    └── Storage is shared — all instances see same data            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Clusters & Instances

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLUSTER → INSTANCE HIERARCHY                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │ CLUSTER: "my-alloydb-cluster"                                 │  │
│ │ (Logical container — holds instances + storage)               │  │
│ │ Region: us-central1                                           │  │
│ │ Network: projects/my-proj/global/networks/default             │  │
│ │ PostgreSQL version: 15                                        │  │
│ │ Storage: auto-scaled, shared across all instances             │  │
│ │                                                               │  │
│ │   ┌─── PRIMARY INSTANCE ────────────────────────────────┐    │  │
│ │   │ Name: "primary"                                      │    │  │
│ │   │ Machine type: 4 vCPU, 32 GB RAM                     │    │  │
│ │   │ Role: Read + Write                                    │    │  │
│ │   │ Zone: us-central1-a                                  │    │  │
│ │   │                                                       │    │  │
│ │   │ (HA enabled → automatic standby in us-central1-b)    │    │  │
│ │   └──────────────────────────────────────────────────────┘    │  │
│ │                                                               │  │
│ │   ┌─── READ POOL ──────────────────────────────────────┐     │  │
│ │   │ Name: "read-pool"                                    │     │  │
│ │   │ Machine type: 2 vCPU, 16 GB RAM                     │     │  │
│ │   │ Node count: 2                                        │     │  │
│ │   │ Role: Read-only (analytics, reporting, read replicas)│     │  │
│ │   │                                                       │     │  │
│ │   │ ┌──────────┐  ┌──────────┐                          │     │  │
│ │   │ │ Node 1   │  │ Node 2   │  ← auto load-balanced    │     │  │
│ │   │ │ Zone -a  │  │ Zone -b  │                          │     │  │
│ │   │ └──────────┘  └──────────┘                          │     │  │
│ │   └──────────────────────────────────────────────────────┘     │  │
│ │                                                               │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Key rules:                                                          │
│ ├── One cluster = one region (regional, not zonal)                 │
│ ├── One primary instance per cluster (required)                    │
│ ├── Zero or more read pool instances per cluster                   │
│ ├── Each read pool can have 1-20 nodes                             │
│ ├── HA standby is automatic when HA is enabled (hidden from you)  │
│ ├── All instances share the same storage layer                     │
│ ├── Cross-region: secondary clusters for DR (separate clusters)    │
│ └── Primary and read pool can have DIFFERENT machine types         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Creating an AlloyDB Cluster

### Console Walkthrough

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → AlloyDB → Create Cluster                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Cluster Type                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ Highly available (recommended ✅)                           │   │
│ │   • Primary + automatic standby in different zone            │   │
│ │   • 99.99% SLA                                               │   │
│ │   • Automatic failover < 60 seconds                          │   │
│ │   • 2x compute cost (standby runs in background)            │   │
│ │                                                               │   │
│ │ ○ Basic                                                       │   │
│ │   • Primary only, no standby                                 │   │
│ │   • No HA SLA                                                │   │
│ │   • For dev/test or budget-constrained workloads             │   │
│ │                                                               │   │
│ │ ○ Highly available with cross-region replication             │   │
│ │   • Primary cluster + secondary cluster in another region    │   │
│ │   • Disaster recovery across regions                         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 2: Cluster Info                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Cluster ID:          [my-alloydb-prod]                        │   │
│ │ Password:            [••••••••••••]                           │   │
│ │   ← PostgreSQL "postgres" user password                      │   │
│ │                                                               │   │
│ │ Database version:    [PostgreSQL 15] ✅                       │   │
│ │   Options: PostgreSQL 14, PostgreSQL 15                      │   │
│ │                                                               │   │
│ │ Region:              [us-central1]                            │   │
│ │   ⚠️ Cannot change after creation!                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 3: Network                                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Network:             [default]                                │   │
│ │ Allocated IP range:  [Auto] or [Select range]                 │   │
│ │                                                               │   │
│ │ ⚠️ AlloyDB uses PRIVATE IP only (no public IP).              │   │
│ │    Requires Private Services Access configured on the VPC.   │   │
│ │                                                               │   │
│ │ If not set up, click "Set up connection":                    │   │
│ │ 1. Allocate IP range: e.g., 10.100.0.0/16                   │   │
│ │ 2. Create private connection to Google services              │   │
│ │ 3. Wait ~2 minutes for peering to complete                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 4: Primary Instance                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Instance ID:         [primary]                                │   │
│ │                                                               │   │
│ │ Machine type:                                                 │   │
│ │ ┌──────────────────────────────────────────────────────────┐ │   │
│ │ │ vCPUs │ Memory  │ Best for                               │ │   │
│ │ ├───────┼─────────┼────────────────────────────────────────┤ │   │
│ │ │ 2     │ 16 GB   │ Dev/test, small workloads              │ │   │
│ │ │ 4     │ 32 GB   │ Small production ✅                     │ │   │
│ │ │ 8     │ 64 GB   │ Medium production                      │ │   │
│ │ │ 16    │ 128 GB  │ Large production                       │ │   │
│ │ │ 32    │ 256 GB  │ Heavy OLTP                             │ │   │
│ │ │ 64    │ 512 GB  │ Enterprise / high-concurrency          │ │   │
│ │ │ 96    │ 768 GB  │ Extreme workloads                      │ │   │
│ │ └──────────────────────────────────────────────────────────┘ │   │
│ │                                                               │   │
│ │ Machine family:                                               │   │
│ │ ○ General purpose (recommended ✅)                            │   │
│ │ ○ Memory optimized (for large in-memory datasets)            │   │
│ │                                                               │   │
│ │ Flags (PostgreSQL parameters):                                │   │
│ │ [+ Add Flag]                                                  │   │
│ │ e.g., max_connections = 500                                  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 5: Encryption (optional)                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ○ Google-managed encryption key (default) ✅                  │   │
│ │ ○ Customer-managed encryption key (CMEK)                      │   │
│ │   → Cloud KMS key in same region                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 6: Automated Backups                                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ☑ Enable automated backups ✅                                 │   │
│ │ Retention: [14] days                                          │   │
│ │ Backup window: [Automatic] or [Custom time]                  │   │
│ │                                                               │   │
│ │ ☑ Enable continuous backup (PITR) ✅                          │   │
│ │ Retention: [14] days                                          │   │
│ │ WAL logs are streamed continuously for point-in-time restore │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ [CREATE CLUSTER]                                                     │
│                                                                       │
│ ⚡ Cluster creation takes ~5-10 minutes.                            │
│ ⚡ Primary instance starts after storage layer is ready.            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Instance Types — Primary & Read Pool

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIMARY INSTANCE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ • ONE primary per cluster (required)                                │
│ • Handles ALL writes and can serve reads                           │
│ • Runs PostgreSQL engine with AlloyDB extensions                   │
│ • When HA is enabled, an invisible standby is maintained            │
│                                                                       │
│ Capabilities:                                                       │
│ ├── Full read/write SQL access                                     │
│ ├── DDL operations (CREATE TABLE, ALTER, etc.)                     │
│ ├── Extension management (CREATE EXTENSION)                        │
│ ├── User/role management                                            │
│ ├── Database flags configuration                                   │
│ └── Columnar engine management                                     │
│                                                                       │
│ Scaling:                                                            │
│ ├── Vertical: Change machine type (requires brief restart)         │
│ └── Cannot horizontally scale the primary                          │
│     (use read pool for horizontal read scaling)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           READ POOL INSTANCES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Read pool = group of identically configured read-only instances    │
│ that serve read traffic from the same storage layer.               │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Application                                                  │   │
│ │       │                                                       │   │
│ │       │  Reads                                                │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────────────────────────┐                    │   │
│ │  │  Read Pool IP (single endpoint)       │                    │   │
│ │  │  → Load-balanced across nodes         │                    │   │
│ │  └────────────┬─────────────────────────┘                    │   │
│ │       ┌───────┼───────┐                                      │   │
│ │       ▼       ▼       ▼                                      │   │
│ │  ┌────────┐┌────────┐┌────────┐                              │   │
│ │  │ Node 1 ││ Node 2 ││ Node 3 │                              │   │
│ │  │2 vCPU  ││2 vCPU  ││2 vCPU  │                              │   │
│ │  │16 GB   ││16 GB   ││16 GB   │                              │   │
│ │  │Zone -a ││Zone -b ││Zone -c │                              │   │
│ │  └────────┘└────────┘└────────┘                              │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Key characteristics:                                                │
│ ├── Read-only (SELECT only — no INSERT/UPDATE/DELETE)              │
│ ├── 1-20 nodes per read pool                                       │
│ ├── Automatic load balancing via single read endpoint              │
│ ├── Each node has its own ultra-fast cache                         │
│ ├── Different machine type from primary is OK                      │
│ │   (e.g., primary = 16 vCPU, read pool = 2 vCPU per node)       │
│ ├── Spread across zones for HA                                     │
│ ├── Near-zero replication lag (shared storage, WAL streaming)      │
│ └── Columnar engine works on read pool nodes too                   │
│                                                                       │
│ Multiple read pools per cluster:                                   │
│ ├── "analytics-pool" — 4 nodes × 8 vCPU (heavy analytics)        │
│ ├── "api-pool" — 2 nodes × 2 vCPU (low-latency API reads)        │
│ └── Each pool has its own connection endpoint                      │
│                                                                       │
│ Use cases:                                                          │
│ ├── Offload read-heavy queries from primary                        │
│ ├── Separate analytics queries from transactional workload         │
│ ├── Scale read throughput horizontally                              │
│ └── Enable columnar engine on dedicated analytics nodes            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Connecting to AlloyDB

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECTING TO ALLOYDB                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ AlloyDB = Private IP ONLY (no public endpoint).                    │
│                                                                       │
│ Three ways to connect:                                              │
│                                                                       │
│ 1. DIRECT PRIVATE IP (from same VPC)                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # From a GCE VM or GKE pod in the same VPC:                  │   │
│ │ psql -h 10.100.0.5 -U postgres -d mydb                       │   │
│ │                                                               │   │
│ │ # Python                                                      │   │
│ │ import psycopg2                                               │   │
│ │ conn = psycopg2.connect(                                      │   │
│ │     host="10.100.0.5",                                        │   │
│ │     database="mydb",                                          │   │
│ │     user="postgres",                                          │   │
│ │     password="your-password"                                  │   │
│ │ )                                                             │   │
│ │                                                               │   │
│ │ ✅ Simplest, lowest latency                                   │   │
│ │ ⚠️ Must be in same VPC (or peered VPC)                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. AUTH PROXY (recommended for most apps)                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ The AlloyDB Auth Proxy handles:                               │   │
│ │ • Automatic TLS encryption                                   │   │
│ │ • IAM-based authentication (no password management)          │   │
│ │ • Secure tunnel from any network to AlloyDB                  │   │
│ │                                                               │   │
│ │ # Download Auth Proxy                                         │   │
│ │ wget https://storage.googleapis.com/alloydb-auth-proxy/v1/    │   │
│ │   alloydb-auth-proxy.linux.amd64 -O alloydb-auth-proxy       │   │
│ │ chmod +x alloydb-auth-proxy                                   │   │
│ │                                                               │   │
│ │ # Run proxy (connects via localhost:5432)                     │   │
│ │ ./alloydb-auth-proxy \                                        │   │
│ │   "projects/my-proj/locations/us-central1/clusters/\         │   │
│ │     my-alloydb-prod/instances/primary"                        │   │
│ │                                                               │   │
│ │ # Now connect via localhost                                   │   │
│ │ psql -h 127.0.0.1 -U postgres -d mydb                        │   │
│ │                                                               │   │
│ │ ✅ Works from anywhere (local dev, Cloud Run, GKE)           │   │
│ │ ✅ Automatic IAM auth and TLS                                │   │
│ │ ✅ Sidecar pattern for GKE                                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. LANGUAGE CONNECTORS (recommended for Cloud Run / Functions)     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Official connectors handle auth and TLS in-process:          │   │
│ │                                                               │   │
│ │ # Python                                                      │   │
│ │ pip install "cloud-sql-python-connector[pg8000]"              │   │
│ │ # (uses same connector library as Cloud SQL)                 │   │
│ │                                                               │   │
│ │ from google.cloud.alloydb.connector import Connector          │   │
│ │ import sqlalchemy                                             │   │
│ │                                                               │   │
│ │ connector = Connector()                                       │   │
│ │                                                               │   │
│ │ def getconn():                                                │   │
│ │     return connector.connect(                                 │   │
│ │         "projects/my-proj/locations/us-central1/"             │   │
│ │         "clusters/my-alloydb-prod/instances/primary",         │   │
│ │         "pg8000",                                             │   │
│ │         user="postgres",                                      │   │
│ │         password="your-password",                             │   │
│ │         db="mydb",                                            │   │
│ │     )                                                         │   │
│ │                                                               │   │
│ │ engine = sqlalchemy.create_engine(                            │   │
│ │     "postgresql+pg8000://",                                   │   │
│ │     creator=getconn,                                          │   │
│ │ )                                                             │   │
│ │                                                               │   │
│ │ Available connectors:                                         │   │
│ │ ├── Python (cloud-sql-python-connector)                      │   │
│ │ ├── Java (alloydb-java-connector)                            │   │
│ │ ├── Go (alloydb-go-connector)                                │   │
│ │ └── Node.js (alloydb-nodejs-connector)                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Connection to read pool:                                            │
│ ├── Use the read pool's separate IP address                        │
│ ├── Or separate Auth Proxy instance string for read pool           │
│ ├── Application routes reads to read pool, writes to primary       │
│ └── Some ORMs support read/write splitting natively               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Columnar Engine

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALLOYDB COLUMNAR ENGINE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The killer feature that sets AlloyDB apart from Cloud SQL.         │
│ Automatically accelerates analytical queries by up to 100x.        │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Regular PostgreSQL storage:        Columnar engine:          │   │
│ │  (row-oriented)                     (column-oriented)        │   │
│ │                                                               │   │
│ │  ┌────┬────┬────┬────┐             ┌────┬────┬────┬────┐    │   │
│ │  │ id │name│city│sale│             │ id │ id │ id │ id │    │   │
│ │  ├────┼────┼────┼────┤             │  1 │  2 │  3 │  4 │    │   │
│ │  │  1 │ A  │ NY │ 50 │             ├────┼────┼────┼────┤    │   │
│ │  │  2 │ B  │ LA │ 30 │             │name│name│name│name│    │   │
│ │  │  3 │ C  │ NY │ 80 │             │ A  │ B  │ C  │ D  │    │   │
│ │  │  4 │ D  │ SF │ 20 │             ├────┼────┼────┼────┤    │   │
│ │  └────┴────┴────┴────┘             │sale│sale│sale│sale│    │   │
│ │                                     │ 50 │ 30 │ 80 │ 20 │    │   │
│ │  Query: SELECT SUM(sale)           └────┴────┴────┴────┘    │   │
│ │         FROM orders                                          │   │
│ │         WHERE city = 'NY';         Only reads "city" and     │   │
│ │                                     "sale" columns. Skips     │   │
│ │  Reads ALL columns for             "id" and "name" entirely. │   │
│ │  each row. Wastes I/O.             Much less I/O = faster.   │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Key features:                                                       │
│ ├── AUTOMATIC: AlloyDB recommends which columns to store columnar │
│ ├── No schema changes needed (same tables, same queries)          │
│ ├── Transparent to the application (query optimizer decides)       │
│ ├── Works on primary AND read pool instances                       │
│ ├── Stored in the ultra-fast cache (local SSD)                    │
│ ├── Always up-to-date (auto-refreshed from row store)             │
│ └── Can manually specify columns or let AI recommend              │
│                                                                       │
│ Enable and configure:                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ -- Check columnar engine status                               │   │
│ │ SHOW google_columnar_engine.enabled;                          │   │
│ │                                                               │   │
│ │ -- Enable columnar engine (database flag)                     │   │
│ │ -- Set via Console or gcloud:                                 │   │
│ │ -- google_columnar_engine.enabled = on                        │   │
│ │                                                               │   │
│ │ -- See auto-recommended columns                               │   │
│ │ SELECT * FROM                                                 │   │
│ │   google_columnar_engine_recommendations();                   │   │
│ │                                                               │   │
│ │ -- Manually add specific columns to columnar store            │   │
│ │ SELECT google_columnar_engine_add(                            │   │
│ │   'public',    -- schema                                      │   │
│ │   'orders',    -- table                                       │   │
│ │   'sale'       -- column                                      │   │
│ │ );                                                            │   │
│ │                                                               │   │
│ │ -- Check which columns are in columnar store                  │   │
│ │ SELECT * FROM g_columnar_columns;                             │   │
│ │                                                               │   │
│ │ -- Verify a query uses columnar engine (look for "columnar") │   │
│ │ EXPLAIN (ANALYZE, COSTS, VERBOSE)                             │   │
│ │   SELECT SUM(sale) FROM orders WHERE city = 'NY';             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Best suited for:                                                    │
│ ├── Aggregations: SUM, AVG, COUNT, MIN, MAX                       │
│ ├── Analytical JOINs on large tables                               │
│ ├── WHERE clauses scanning many rows                               │
│ ├── GROUP BY queries                                                │
│ └── Reporting and dashboards                                        │
│                                                                       │
│ NOT helpful for:                                                    │
│ ├── Point lookups (SELECT ... WHERE id = 123) — row store is fine │
│ ├── Small tables (< 1000 rows)                                     │
│ └── INSERT/UPDATE/DELETE (only accelerates reads)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: High Availability & Failover

```
┌─────────────────────────────────────────────────────────────────────┐
│           HIGH AVAILABILITY                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ HA Cluster (recommended for production):                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Zone A                      Zone B                           │   │
│ │  ┌──────────────────┐       ┌──────────────────┐             │   │
│ │  │ PRIMARY          │  WAL  │ STANDBY          │             │   │
│ │  │                  │──────→│ (automatic)      │             │   │
│ │  │ Read + Write     │ sync  │ Hot standby      │             │   │
│ │  │ Application      │ repl  │ Invisible to     │             │   │
│ │  │ connects here    │       │ applications     │             │   │
│ │  │                  │       │                  │             │   │
│ │  │ Ultra-fast cache │       │ Ultra-fast cache │             │   │
│ │  └──────────────────┘       └──────────────────┘             │   │
│ │           │                           │                       │   │
│ │           └───────────┬───────────────┘                       │   │
│ │                       ▼                                       │   │
│ │            ┌──────────────────┐                               │   │
│ │            │ Shared Storage   │                               │   │
│ │            │ (3x replicated)  │                               │   │
│ │            └──────────────────┘                               │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Failover behavior:                                                  │
│ ├── Detection: AlloyDB monitors the primary continuously           │
│ ├── Trigger: Primary becomes unresponsive                          │
│ ├── Failover time: < 60 seconds typically                          │
│ ├── Process:                                                       │
│ │   1. Standby promoted to primary                                │
│ │   2. DNS updated to point to new primary                        │
│ │   3. New standby created automatically                          │
│ │   4. Read pool instances reconnect automatically                │
│ ├── Data loss: Zero (synchronous replication to standby)           │
│ └── Application impact: Brief connection drop, must reconnect     │
│                                                                       │
│ ⚠️ HA doubles your compute cost (standby = same machine type).    │
│ ⚠️ Standby is NOT accessible for reads (unlike read pool).        │
│ ⚠️ Manual failover available for testing:                          │
│    gcloud alloydb instances failover primary \                     │
│      --cluster=my-alloydb-prod --region=us-central1                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Cross-Region Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│           CROSS-REGION REPLICATION                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ For disaster recovery across regions, AlloyDB supports             │
│ secondary clusters in different regions.                            │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Primary Region: us-central1      Secondary Region: us-east1 │   │
│ │                                                               │   │
│ │  ┌────────────────────┐          ┌────────────────────┐      │   │
│ │  │ PRIMARY CLUSTER    │   async  │ SECONDARY CLUSTER  │      │   │
│ │  │                    │   WAL    │                    │      │   │
│ │  │ Primary instance   │─────────→│ Secondary instance │      │   │
│ │  │ (read/write)       │   repl   │ (read-only)        │      │   │
│ │  │                    │          │                    │      │   │
│ │  │ Read pool (opt.)   │          │ Read pool (opt.)   │      │   │
│ │  │                    │          │                    │      │   │
│ │  │ ┌──────────────┐  │          │ ┌──────────────┐  │      │   │
│ │  │ │   Storage    │  │          │ │   Storage    │  │      │   │
│ │  │ │   (primary)  │  │          │ │   (replica)  │  │      │   │
│ │  │ └──────────────┘  │          │ └──────────────┘  │      │   │
│ │  └────────────────────┘          └────────────────────┘      │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Key characteristics:                                                │
│ ├── Asynchronous replication (not synchronous — some lag)          │
│ ├── Replication lag: typically seconds, depends on write volume    │
│ ├── Secondary cluster serves read traffic (offload reads globally)│
│ ├── Promotion: secondary → primary (manual, during DR event)     │
│ ├── Promotion is one-way (promoted cluster becomes independent)   │
│ ├── Can add read pool instances to secondary cluster              │
│ └── Columnar engine works on secondary cluster too                │
│                                                                       │
│ DR failover process:                                               │
│ 1. Primary region fails or planned migration                      │
│ 2. Promote secondary cluster:                                      │
│    gcloud alloydb clusters promote my-secondary-cluster \          │
│      --region=us-east1                                             │
│ 3. Secondary becomes independent primary (read/write)              │
│ 4. Update DNS / application config to point to new cluster        │
│ 5. Re-create replication in opposite direction (if desired)        │
│                                                                       │
│ ⚠️ After promotion, replication link is broken.                    │
│    Must set up new replication if you want cross-region again.    │
│ ⚠️ Some data loss possible (RPO = replication lag at time of DR). │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Backups & Restore

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKUPS & POINT-IN-TIME RECOVERY                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. AUTOMATED BACKUPS                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ • Enabled by default when creating a cluster                  │   │
│ │ • Daily full backup (automated)                               │   │
│ │ • Retention: configurable, 1-365 days (default 14 days)      │   │
│ │ • Backup window: configurable or automatic                   │   │
│ │ • Stored in same region as cluster                           │   │
│ │ • Encryption: same as cluster (Google-managed or CMEK)       │   │
│ │                                                               │   │
│ │ gcloud alloydb backups list --region=us-central1              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. ON-DEMAND BACKUPS                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Create manual backup                                        │   │
│ │ gcloud alloydb backups create my-backup-20240115 \            │   │
│ │   --cluster=my-alloydb-prod \                                 │   │
│ │   --region=us-central1                                        │   │
│ │                                                               │   │
│ │ • Not automatically deleted (manual lifecycle management)    │   │
│ │ • Useful before major changes or migrations                  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. CONTINUOUS BACKUP / PITR                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ WAL logs are streamed continuously, enabling restore to any  │   │
│ │ point in time within the retention window.                    │   │
│ │                                                               │   │
│ │ Retention: 1-35 days (default 14 days)                       │   │
│ │                                                               │   │
│ │ # Restore to specific point in time                          │   │
│ │ gcloud alloydb clusters restore my-restored-cluster \         │   │
│ │   --source-cluster=my-alloydb-prod \                          │   │
│ │   --point-in-time="2024-01-15T10:30:00Z" \                   │   │
│ │   --region=us-central1 \                                      │   │
│ │   --network=default                                           │   │
│ │                                                               │   │
│ │ ⚠️ PITR creates a NEW cluster (does not overwrite existing). │   │
│ │ ⚠️ New cluster needs new primary instance.                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. RESTORE FROM BACKUP                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Restore creates a new cluster from backup                   │   │
│ │ gcloud alloydb clusters restore my-restored-cluster \         │   │
│ │   --backup=projects/my-proj/locations/us-central1/\          │   │
│ │     backups/my-backup-20240115 \                              │   │
│ │   --region=us-central1 \                                      │   │
│ │   --network=default                                           │   │
│ │                                                               │   │
│ │ After restoring the cluster, create a primary instance:      │   │
│ │ gcloud alloydb instances create primary \                     │   │
│ │   --cluster=my-restored-cluster \                             │   │
│ │   --region=us-central1 \                                      │   │
│ │   --instance-type=PRIMARY \                                   │   │
│ │   --cpu-count=4                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Backup comparison:                                                  │
│ ┌──────────────────┬───────────────┬──────────────┬──────────────┐│
│ │ Type             │ Automated     │ On-Demand    │ PITR         ││
│ ├──────────────────┼───────────────┼──────────────┼──────────────┤│
│ │ Trigger          │ Daily (auto)  │ Manual       │ Continuous   ││
│ │ Retention        │ 1-365 days    │ Until deleted│ 1-35 days    ││
│ │ Granularity      │ Daily         │ Point-in-time│ Any second   ││
│ │ Restore creates  │ New cluster   │ New cluster  │ New cluster  ││
│ │ Cost             │ Backup storage│ Backup storage│ WAL storage ││
│ └──────────────────┴───────────────┴──────────────┴──────────────┘│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Maintenance & Updates

```
┌─────────────────────────────────────────────────────────────────────┐
│           MAINTENANCE & VERSION MANAGEMENT                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Maintenance windows:                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console → AlloyDB → Cluster → Maintenance                    │   │
│ │                                                               │   │
│ │ • Day: [Any / specific weekday]                              │   │
│ │ • Window: [1-hour window, e.g., 03:00-04:00 UTC]            │   │
│ │                                                               │   │
│ │ HA clusters: Failover-based maintenance (near-zero downtime) │   │
│ │ ├── 1. Patch standby first                                   │   │
│ │ ├── 2. Failover to patched standby                          │   │
│ │ ├── 3. Patch old primary (now standby)                      │   │
│ │ └── Brief connection interruption during failover            │   │
│ │                                                               │   │
│ │ Basic clusters: Brief downtime during maintenance.           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ PostgreSQL version upgrades:                                        │
│ ├── Minor versions: automatic (part of maintenance)                │
│ ├── Major versions (14 → 15): manual, in-place upgrade            │
│ │   gcloud alloydb clusters upgrade my-alloydb-prod \             │
│ │     --region=us-central1 \                                       │
│ │     --version=POSTGRES_15                                        │
│ │   ⚠️ Requires brief downtime                                    │
│ │   ⚠️ Test in staging first                                      │
│ └── Rollback: Not supported — take backup before upgrade          │
│                                                                       │
│ Database flags (PostgreSQL parameters):                             │
│ ├── Set via Console or gcloud                                      │
│ ├── Some require restart, some are dynamic                         │
│ ├── Examples:                                                       │
│ │   ├── max_connections = 500                                     │
│ │   ├── work_mem = 64MB                                           │
│ │   ├── shared_preload_libraries = 'pg_stat_statements'           │
│ │   └── google_columnar_engine.enabled = on                       │
│ └── gcloud alloydb instances update primary \                     │
│       --cluster=my-alloydb-prod \                                  │
│       --region=us-central1 \                                       │
│       --database-flags=max_connections=500,work_mem=67108864       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Security & IAM

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALLOYDB SECURITY                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. NETWORK SECURITY                                                │
│ ├── Private IP ONLY — no public internet access                    │
│ ├── Deployed via Private Services Access (VPC peering)             │
│ ├── Firewall rules control which subnets/VMs can connect           │
│ ├── VPC Service Controls: restrict AlloyDB to service perimeter    │
│ └── Auth Proxy: secure tunnel for external access                  │
│                                                                       │
│ 2. AUTHENTICATION                                                  │
│ ├── PostgreSQL native auth (username/password)                     │
│ │   CREATE USER app_user WITH PASSWORD 'strong-password';         │
│ ├── IAM database authentication:                                   │
│ │   ├── Use Google Cloud IAM principals as PostgreSQL users       │
│ │   ├── No password management — uses OAuth2 tokens               │
│ │   ├── Grant IAM role: roles/alloydb.databaseUser                │
│ │   └── Connect via Auth Proxy with --auto-iam-authn flag         │
│ └── Service account auth (recommended for applications)            │
│                                                                       │
│ 3. IAM ROLES                                                       │
│ ┌──────────────────────────────┬──────────────────────────────────┐│
│ │ Role                         │ Permissions                       ││
│ ├──────────────────────────────┼──────────────────────────────────┤│
│ │ alloydb.admin                │ Full admin: clusters, instances, ││
│ │                              │ backups, users, IAM               ││
│ │ alloydb.databaseUser         │ Connect and query (IAM auth)     ││
│ │ alloydb.viewer               │ Read-only metadata (no data)     ││
│ │ alloydb.client               │ Connect via Auth Proxy           ││
│ └──────────────────────────────┴──────────────────────────────────┘│
│                                                                       │
│ 4. ENCRYPTION                                                       │
│ ├── At rest: Always encrypted (Google-managed or CMEK)             │
│ ├── In transit: TLS enforced (via Auth Proxy or direct SSL)        │
│ ├── Backups: Encrypted with same key as cluster                    │
│ └── CMEK rotation: Automatic (Cloud KMS handles rotation)          │
│                                                                       │
│ 5. AUDIT LOGGING                                                    │
│ ├── Admin Activity logs: enabled by default (cluster operations)   │
│ ├── Data Access logs: opt-in (SQL queries — can be verbose)       │
│ ├── pgAudit extension: PostgreSQL-level audit logging              │
│ │   CREATE EXTENSION pgaudit;                                     │
│ │   SET pgaudit.log = 'write, ddl';                               │
│ └── Logs available in Cloud Logging                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Monitoring & Performance

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING ALLOYDB                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Built-in monitoring (Console → AlloyDB → Instance → Monitoring):  │
│                                                                       │
│ Key metrics:                                                        │
│ ┌────────────────────────────┬──────────────────────────────────┐  │
│ │ Metric                     │ Alert threshold                    │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ CPU utilization            │ > 80% sustained → scale up       │  │
│ │ Memory utilization         │ > 85% → increase instance size   │  │
│ │ Active connections         │ > 80% of max_connections          │  │
│ │ Storage used               │ Monitor growth trend              │  │
│ │ Transaction rate (TPS)     │ Baseline awareness                │  │
│ │ Query latency (p50/p99)    │ > 2x baseline → investigate      │  │
│ │ Replication lag (read pool)│ > 1 sec sustained → investigate  │  │
│ │ Disk I/O (read/write)      │ Approaching limits               │  │
│ │ Deadlocks                  │ > 0 → investigate queries        │  │
│ │ Rows returned/scanned      │ High ratio = missing indexes     │  │
│ └────────────────────────────┴──────────────────────────────────┘  │
│                                                                       │
│ Query Insights (built-in):                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console → AlloyDB → Instance → Query Insights               │   │
│ │                                                               │   │
│ │ • Top queries by execution time                              │   │
│ │ • Query plan visualization                                   │   │
│ │ • Wait events analysis                                       │   │
│ │ • Normalized query fingerprints                              │   │
│ │ • Filter by database, user, time range                       │   │
│ │                                                               │   │
│ │ ⚡ No pg_stat_statements setup needed — built-in!            │   │
│ │ ⚡ Shows query plans without running EXPLAIN manually.       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Performance tuning tips:                                            │
│ ├── Enable columnar engine for analytical queries                  │
│ ├── Use read pool to offload read traffic from primary             │
│ ├── Properly size instances (CPU/memory match workload)            │
│ ├── Use connection pooling (PgBouncer or application-level)        │
│ ├── Create appropriate indexes (B-tree, GIN, GiST as needed)      │
│ ├── Vacuum regularly (autovacuum should be configured)             │
│ ├── Monitor long-running transactions (they block vacuuming)       │
│ └── Use EXPLAIN ANALYZE to identify slow query plans               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Terraform & CLI

### Terraform

```hcl
# ─────────────────────────────────────────────────────────────
# Private Services Access (required for AlloyDB)
# ─────────────────────────────────────────────────────────────

resource "google_compute_global_address" "private_ip_range" {
  name          = "alloydb-private-range"
  project       = "my-gcp-project"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = "projects/my-gcp-project/global/networks/default"
}

resource "google_service_networking_connection" "private_vpc" {
  network                 = "projects/my-gcp-project/global/networks/default"
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_range.name]
}

# ─────────────────────────────────────────────────────────────
# AlloyDB Cluster (HA)
# ─────────────────────────────────────────────────────────────

resource "google_alloydb_cluster" "main" {
  cluster_id = "my-alloydb-prod"
  location   = "us-central1"
  project    = "my-gcp-project"
  network_config {
    network = "projects/my-gcp-project/global/networks/default"
  }

  database_version = "POSTGRES_15"

  # Automated backups
  automated_backup_policy {
    enabled = true

    backup_window = "3600s"  # 1 hour window

    weekly_schedule {
      days_of_week = ["MONDAY", "WEDNESDAY", "FRIDAY"]
      start_times {
        hours   = 3
        minutes = 0
      }
    }

    quantity_based_retention {
      count = 14
    }
  }

  # Continuous backup (PITR)
  continuous_backup_config {
    enabled              = true
    recovery_window_days = 14
  }

  initial_user {
    user     = "postgres"
    password = var.alloydb_password  # Use a variable, not hardcoded!
  }

  labels = {
    environment = "production"
    team        = "backend"
  }

  depends_on = [google_service_networking_connection.private_vpc]
}

# ─────────────────────────────────────────────────────────────
# Primary Instance
# ─────────────────────────────────────────────────────────────

resource "google_alloydb_instance" "primary" {
  cluster       = google_alloydb_cluster.main.name
  instance_id   = "primary"
  instance_type = "PRIMARY"

  machine_config {
    cpu_count = 4  # 4 vCPU = 32 GB RAM (auto-determined)
  }

  database_flags = {
    "max_connections"                    = "500"
    "google_columnar_engine.enabled"     = "on"
    "shared_preload_libraries"           = "pg_stat_statements"
  }

  labels = {
    role = "primary"
  }
}

# ─────────────────────────────────────────────────────────────
# Read Pool
# ─────────────────────────────────────────────────────────────

resource "google_alloydb_instance" "read_pool" {
  cluster       = google_alloydb_cluster.main.name
  instance_id   = "read-pool"
  instance_type = "READ_POOL"

  machine_config {
    cpu_count = 2  # 2 vCPU = 16 GB RAM per node
  }

  read_pool_config {
    node_count = 2
  }

  database_flags = {
    "google_columnar_engine.enabled" = "on"
  }

  labels = {
    role = "analytics"
  }

  depends_on = [google_alloydb_instance.primary]
}

# ─────────────────────────────────────────────────────────────
# Secondary Cluster (Cross-Region DR)
# ─────────────────────────────────────────────────────────────

resource "google_alloydb_cluster" "secondary" {
  cluster_id   = "my-alloydb-secondary"
  location     = "us-east1"
  project      = "my-gcp-project"
  cluster_type = "SECONDARY"

  network_config {
    network = "projects/my-gcp-project/global/networks/default"
  }

  secondary_config {
    primary_cluster_name = google_alloydb_cluster.main.name
  }

  depends_on = [google_alloydb_instance.primary]
}

resource "google_alloydb_instance" "secondary_primary" {
  cluster       = google_alloydb_cluster.secondary.name
  instance_id   = "secondary-instance"
  instance_type = "SECONDARY"

  machine_config {
    cpu_count = 4
  }
}

# ─────────────────────────────────────────────────────────────
# IAM Binding
# ─────────────────────────────────────────────────────────────

resource "google_project_iam_member" "alloydb_client" {
  project = "my-gcp-project"
  role    = "roles/alloydb.client"
  member  = "serviceAccount:my-app@my-gcp-project.iam.gserviceaccount.com"
}
```

### gcloud CLI Reference

```bash
# ─────────────────────────────────────────────────────────────
# Cluster Management
# ─────────────────────────────────────────────────────────────

# Create HA cluster
gcloud alloydb clusters create my-alloydb-prod \
  --region=us-central1 \
  --network=default \
  --password=YOUR_SECURE_PASSWORD \
  --database-version=POSTGRES_15

# List clusters
gcloud alloydb clusters list --region=us-central1

# Describe cluster
gcloud alloydb clusters describe my-alloydb-prod \
  --region=us-central1

# Update cluster (backup policy)
gcloud alloydb clusters update my-alloydb-prod \
  --region=us-central1 \
  --automated-backup-enabled \
  --automated-backup-days-of-week=MONDAY,WEDNESDAY,FRIDAY \
  --automated-backup-start-times=03:00 \
  --automated-backup-retention-count=14

# Delete cluster (must delete instances first)
gcloud alloydb clusters delete my-alloydb-prod \
  --region=us-central1

# ─────────────────────────────────────────────────────────────
# Instance Management
# ─────────────────────────────────────────────────────────────

# Create primary instance
gcloud alloydb instances create primary \
  --cluster=my-alloydb-prod \
  --region=us-central1 \
  --instance-type=PRIMARY \
  --cpu-count=4 \
  --database-flags=max_connections=500,\
google_columnar_engine.enabled=on

# Create read pool
gcloud alloydb instances create read-pool \
  --cluster=my-alloydb-prod \
  --region=us-central1 \
  --instance-type=READ_POOL \
  --cpu-count=2 \
  --read-pool-node-count=2

# List instances
gcloud alloydb instances list \
  --cluster=my-alloydb-prod \
  --region=us-central1

# Describe instance (shows IP address)
gcloud alloydb instances describe primary \
  --cluster=my-alloydb-prod \
  --region=us-central1

# Scale primary (change machine type)
gcloud alloydb instances update primary \
  --cluster=my-alloydb-prod \
  --region=us-central1 \
  --cpu-count=8

# Scale read pool (add nodes)
gcloud alloydb instances update read-pool \
  --cluster=my-alloydb-prod \
  --region=us-central1 \
  --read-pool-node-count=4

# Update database flags
gcloud alloydb instances update primary \
  --cluster=my-alloydb-prod \
  --region=us-central1 \
  --database-flags=max_connections=1000,work_mem=67108864

# Trigger manual failover (HA clusters)
gcloud alloydb instances failover primary \
  --cluster=my-alloydb-prod \
  --region=us-central1

# Delete instance
gcloud alloydb instances delete read-pool \
  --cluster=my-alloydb-prod \
  --region=us-central1

# ─────────────────────────────────────────────────────────────
# Backup Management
# ─────────────────────────────────────────────────────────────

# Create on-demand backup
gcloud alloydb backups create my-backup-20240115 \
  --cluster=my-alloydb-prod \
  --region=us-central1

# List backups
gcloud alloydb backups list --region=us-central1

# Describe backup
gcloud alloydb backups describe my-backup-20240115 \
  --region=us-central1

# Restore from backup (creates new cluster)
gcloud alloydb clusters restore my-restored-cluster \
  --backup=projects/my-proj/locations/us-central1/\
backups/my-backup-20240115 \
  --region=us-central1 \
  --network=default

# Restore PITR (point-in-time)
gcloud alloydb clusters restore my-pitr-cluster \
  --source-cluster=my-alloydb-prod \
  --point-in-time="2024-01-15T10:30:00Z" \
  --region=us-central1 \
  --network=default

# Delete backup
gcloud alloydb backups delete my-backup-20240115 \
  --region=us-central1

# ─────────────────────────────────────────────────────────────
# Cross-Region Replication
# ─────────────────────────────────────────────────────────────

# Create secondary cluster
gcloud alloydb clusters create my-alloydb-secondary \
  --region=us-east1 \
  --secondary \
  --primary-cluster=projects/my-proj/locations/us-central1/\
clusters/my-alloydb-prod

# Create secondary instance
gcloud alloydb instances create secondary-instance \
  --cluster=my-alloydb-secondary \
  --region=us-east1 \
  --instance-type=SECONDARY \
  --cpu-count=4

# Promote secondary (during DR)
gcloud alloydb clusters promote my-alloydb-secondary \
  --region=us-east1

# ─────────────────────────────────────────────────────────────
# Auth Proxy
# ─────────────────────────────────────────────────────────────

# Download Auth Proxy
wget https://storage.googleapis.com/alloydb-auth-proxy/v1/\
alloydb-auth-proxy.linux.amd64 -O alloydb-auth-proxy
chmod +x alloydb-auth-proxy

# Run Auth Proxy
./alloydb-auth-proxy \
  "projects/my-proj/locations/us-central1/clusters/\
my-alloydb-prod/instances/primary"

# Run with IAM authentication
./alloydb-auth-proxy \
  --auto-iam-authn \
  "projects/my-proj/locations/us-central1/clusters/\
my-alloydb-prod/instances/primary"
```

---

## Part 15: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: SAAS APPLICATION (OLTP + ANALYTICS)              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: SaaS app needing fast transactions + dashboard analytics │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌────────────┐     ┌──────────────────────────┐   │
│ │ Cloud   │────→│ AlloyDB    │────→│ PRIMARY (4 vCPU)         │   │
│ │ Run     │     │ Auth Proxy │     │ Writes + transactional   │   │
│ │ (API)   │     │            │     │ reads                    │   │
│ └─────────┘     └────────────┘     └──────────────────────────┘   │
│                                                                       │
│ ┌─────────┐     ┌────────────┐     ┌──────────────────────────┐   │
│ │ Cloud   │────→│ AlloyDB    │────→│ READ POOL (2 nodes)      │   │
│ │ Run     │     │ Auth Proxy │     │ Columnar engine ON       │   │
│ │ (Dash)  │     │            │     │ Dashboard queries        │   │
│ └─────────┘     └────────────┘     └──────────────────────────┘   │
│                                                                       │
│ Setup:                                                              │
│ ├── HA cluster with PITR enabled                                   │
│ ├── Primary: 4 vCPU for OLTP (writes + point reads)               │
│ ├── Read pool: 2 nodes × 2 vCPU with columnar engine for reports │
│ ├── Application routes: writes → primary, dashboards → read pool │
│ ├── Auth Proxy sidecar in Cloud Run (language connector also OK)  │
│ ├── IAM database auth for service accounts                        │
│ └── Daily automated backups, 14-day PITR                           │
│                                                                       │
│ Cost: ~$650/month (primary $216 + HA $216 + read pool $216)        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: MIGRATION FROM ORACLE / SQL SERVER               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Enterprise migrating from Oracle to PostgreSQL           │
│                                                                       │
│ Architecture:                                                       │
│ ┌────────────┐                      ┌──────────────────────────┐   │
│ │ On-prem    │  Database Migration  │ AlloyDB (HA)             │   │
│ │ Oracle DB  │──────Service────────→│ PRIMARY (16 vCPU)        │   │
│ │            │  (DMS)               │ 128 GB RAM               │   │
│ └────────────┘                      │                          │   │
│                                      │ Read Pool (4 nodes)     │   │
│                                      │ Columnar engine          │   │
│                                      │ CMEK encryption          │   │
│                                      └──────────┬───────────────┘   │
│                                                  │                  │
│                                      ┌───────────┘                  │
│                                      ▼                              │
│                               ┌──────────────┐                     │
│                               │ Secondary    │                     │
│                               │ Cluster      │  ← DR in us-east1  │
│                               │ (us-east1)   │                     │
│                               └──────────────┘                     │
│                                                                       │
│ Migration steps:                                                    │
│ 1. Convert schema: Oracle → PostgreSQL (ora2pg tool)              │
│ 2. Use Database Migration Service (DMS) for data migration        │
│ 3. Validate with AlloyDB read pool (compare query results)        │
│ 4. Enable columnar engine for analytical workloads                 │
│ 5. Cutover: Switch application connection string                   │
│                                                                       │
│ Why AlloyDB over Cloud SQL for Oracle migration:                   │
│ ├── 4x faster transactions handles Oracle-level workloads          │
│ ├── Columnar engine replaces Oracle's in-memory analytics          │
│ ├── 99.99% SLA matches enterprise expectations                     │
│ ├── Cross-region DR matches Oracle Data Guard capabilities         │
│ └── ML integration (Vertex AI) adds capabilities Oracle lacks     │
│                                                                       │
│ Cost: ~$4,000/month (primary $1,728 + HA + read pool + secondary) │
│ Savings vs Oracle: Often 50-80% reduction in license + infra cost │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 3: AI/ML-ENHANCED APPLICATION                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: E-commerce with vector search and AI recommendations    │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌──────────────┐     ┌──────────────────────────┐ │
│ │ Web App │────→│ Cloud Run    │────→│ AlloyDB (HA)             │ │
│ │         │     │ (API)        │     │ PRIMARY (8 vCPU)         │ │
│ │         │     │              │     │                          │ │
│ │ "Show   │     │ SQL query:   │     │ pgvector extension      │ │
│ │ similar │     │ SELECT *     │     │ + Vertex AI integration  │ │
│ │ products│     │ ORDER BY     │     │                          │ │
│ │  to X"  │     │ embedding    │     │ Products table:          │ │
│ │         │     │ <-> $query   │     │ id, name, description,  │ │
│ │         │     │ LIMIT 10     │     │ embedding VECTOR(768)    │ │
│ └─────────┘     └──────────────┘     └──────────────────────────┘ │
│                                                                       │
│ AlloyDB AI features:                                                │
│ ├── pgvector: Store and query vector embeddings in PostgreSQL      │
│ ├── Vertex AI integration:                                         │
│ │   SELECT google_ml.embedding(                                   │
│ │     'textembedding-gecko',                                      │
│ │     'red running shoes'                                          │
│ │   );                                                             │
│ │   → Returns embedding vector directly in SQL!                  │
│ ├── ML predictions from SQL:                                       │
│ │   SELECT google_ml.predict_row(                                 │
│ │     'my-vertex-model',                                          │
│ │     json_build_object('feature1', col1, 'feature2', col2)       │
│ │   ) FROM my_table;                                               │
│ └── No need for separate ML pipeline for simple predictions       │
│                                                                       │
│ Cost: ~$900/month (8 vCPU HA cluster + storage)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALLOYDB QUICK REFERENCE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Fully managed, PostgreSQL-compatible, high-performance DB    │
│ Compatibility: 100% PostgreSQL 14/15                                │
│ Performance: 4x OLTP, 100x OLAP vs standard PostgreSQL             │
│                                                                       │
│ Architecture: Cluster → Primary + Read Pool + Shared Storage      │
│ ├── Primary: 1 per cluster, read/write, 2-96 vCPU                 │
│ ├── Read Pool: 0+ pools, 1-20 nodes each, read-only               │
│ ├── Storage: Auto-scaling, up to 128 TB, log-based                 │
│ └── HA Standby: Automatic, invisible, synchronous replication      │
│                                                                       │
│ Columnar engine:                                                    │
│ ├── Accelerates analytical queries (aggregations, scans)           │
│ ├── Auto-recommends columns to store columnar                      │
│ ├── Transparent to application (optimizer decides)                 │
│ └── Works on primary AND read pool instances                       │
│                                                                       │
│ Connectivity: Private IP ONLY                                      │
│ ├── Direct: From same VPC                                          │
│ ├── Auth Proxy: From anywhere, IAM-based auth                      │
│ └── Language connectors: Python, Java, Go, Node.js                 │
│                                                                       │
│ HA & DR:                                                            │
│ ├── HA: Automatic failover < 60s, 99.99% SLA, zero data loss      │
│ ├── Cross-region: Secondary cluster, async replication              │
│ └── Promotion: Manual, one-way (DR event)                          │
│                                                                       │
│ Backups:                                                            │
│ ├── Automated: Daily, 1-365 day retention                          │
│ ├── PITR: Continuous WAL, 1-35 day window                          │
│ └── Restore: Always creates new cluster                            │
│                                                                       │
│ Min cost: ~$216/month (2 vCPU Basic) | ~$444/month (2 vCPU HA)    │
│                                                                       │
│ vs Cloud SQL: More expensive, much faster, columnar engine,        │
│               ML integration, higher SLA                            │
│ vs Spanner: Regional only, PostgreSQL-native, no global writes,    │
│             better for mixed OLTP+OLAP                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Clusters

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCALING READ POOL NODES FROM CONSOLE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → AlloyDB → Clusters → [your cluster]                      │
│                                                                       │
│ Step 1: Click on your read pool instance name (e.g., "read-pool") │
│ Step 2: Click "Edit" at the top                                    │
│ Step 3: Under "Read pool node count", change the number            │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Read pool node count:  [2] → [4]                              │   │
│ │                                                               │   │
│ │ ⚡ Min: 1 node                                               │   │
│ │ ⚡ Max: 20 nodes                                              │   │
│ │ ⚡ New nodes are automatically spread across zones            │   │
│ │ ⚡ Zero downtime — existing nodes keep serving traffic       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 4: Click "Update Instance"                                    │
│ Step 5: Wait ~2-5 minutes for new nodes to become ready            │
│                                                                       │
│ CLI equivalent:                                                     │
│ gcloud alloydb instances update read-pool \                        │
│   --cluster=my-alloydb-prod \                                      │
│   --region=us-central1 \                                           │
│   --read-pool-node-count=4                                         │
│                                                                       │
│ ⚠️ Scaling DOWN also works — AlloyDB drains connections from       │
│    removed nodes gracefully.                                        │
│ ⚠️ You can also change the machine type (vCPU/RAM) of read pool   │
│    nodes, but that requires a brief restart of affected nodes.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETING INSTANCES FROM CONSOLE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → AlloyDB → Clusters → [your cluster]                      │
│                                                                       │
│ You'll see a list of all instances in the cluster:                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Instance Name   │ Type        │ Status │ Actions             │   │
│ ├─────────────────┼─────────────┼────────┼─────────────────────┤   │
│ │ primary         │ Primary     │ Ready  │ ⋮ (three dots menu) │   │
│ │ read-pool       │ Read Pool   │ Ready  │ ⋮ (three dots menu) │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ To delete a read pool instance:                                    │
│ Step 1: Click the ⋮ menu next to the instance                     │
│ Step 2: Select "Delete"                                            │
│ Step 3: Type the instance name to confirm                          │
│ Step 4: Click "Delete"                                             │
│                                                                       │
│ To delete the primary instance:                                    │
│ Step 1: Delete ALL read pool instances first                       │
│ Step 2: Then click ⋮ on the primary → "Delete"                    │
│ Step 3: Type instance name to confirm                              │
│                                                                       │
│ ⚠️ Deleting the primary instance does NOT delete the cluster.     │
│    The cluster (and its storage/backups) still exists.             │
│ ⚠️ You cannot delete the primary if read pool instances exist.    │
│    Delete read pools FIRST, then primary.                          │
│                                                                       │
│ CLI equivalent:                                                     │
│ # Delete read pool instance                                        │
│ gcloud alloydb instances delete read-pool \                        │
│   --cluster=my-alloydb-prod \                                      │
│   --region=us-central1                                             │
│                                                                       │
│ # Delete primary instance                                          │
│ gcloud alloydb instances delete primary \                          │
│   --cluster=my-alloydb-prod \                                      │
│   --region=us-central1                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETING A CLUSTER FROM CONSOLE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️ You MUST delete ALL instances before deleting the cluster.      │
│    AlloyDB will not let you delete a cluster that has instances.   │
│                                                                       │
│ Deletion order (mandatory):                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │   Step 1: Delete all read pool instances                      │   │
│ │      ↓                                                        │   │
│ │   Step 2: Delete the primary instance                         │   │
│ │      ↓                                                        │   │
│ │   Step 3: Delete the cluster                                  │   │
│ │                                                               │   │
│ │ If you try to skip a step, you'll get an error like:         │   │
│ │ "Cluster has active instances. Delete all instances first."  │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Console steps:                                                      │
│ 1. Console → AlloyDB → Clusters                                   │
│ 2. Delete all instances first (see above section)                  │
│ 3. Click on the cluster name                                       │
│ 4. Click "Delete Cluster" at the top                               │
│ 5. Choose whether to keep or delete backups:                       │
│    ○ "Delete all automated and on-demand backups"                 │
│    ○ "Keep backups" (manual backups are preserved)                │
│ 6. Type the cluster name to confirm                                │
│ 7. Click "Delete"                                                  │
│                                                                       │
│ CLI equivalent:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Step 1: Delete read pool instances                          │   │
│ │ gcloud alloydb instances delete read-pool \                   │   │
│ │   --cluster=my-alloydb-prod \                                 │   │
│ │   --region=us-central1                                        │   │
│ │                                                               │   │
│ │ # Step 2: Delete primary instance                             │   │
│ │ gcloud alloydb instances delete primary \                     │   │
│ │   --cluster=my-alloydb-prod \                                 │   │
│ │   --region=us-central1                                        │   │
│ │                                                               │   │
│ │ # Step 3: Delete the cluster                                  │   │
│ │ gcloud alloydb clusters delete my-alloydb-prod \              │   │
│ │   --region=us-central1                                        │   │
│ │                                                               │   │
│ │ # Optional: Force-delete cluster + all instances at once      │
│ │ gcloud alloydb clusters delete my-alloydb-prod \              │   │
│ │   --region=us-central1 \                                      │   │
│ │   --force                                                     │   │
│ │   # ⚠️ --force deletes instances AND cluster in one command  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Cluster deletion takes ~2-5 minutes.                             │
│ ⚡ Storage is released after cluster deletion.                      │
│ ⚡ Automated backups are deleted unless you chose to keep them.    │
│ ⚡ On-demand (manual) backups are NOT deleted with the cluster     │
│    — you must delete them separately if desired.                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## AlloyDB vs Cloud SQL: When to Choose

```
┌─────────────────────────────────────────────────────────────────────┐
│           SIMPLE DECISION GUIDE                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Both are fully managed PostgreSQL databases on GCP.                │
│ Think of it this way:                                               │
│                                                                       │
│ Cloud SQL = Toyota Camry (reliable, affordable, gets the job done) │
│ AlloyDB   = BMW M5 (high-performance, premium, enterprise-grade)   │
│                                                                       │
│ Both drive you to the same destination (PostgreSQL), but the       │
│ experience and cost are very different.                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Side-by-Side Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALLOYDB vs CLOUD SQL — DETAILED COMPARISON                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────┬───────────────────┬─────────────────────┐ │
│ │ Feature              │ Cloud SQL (PG)    │ AlloyDB              │ │
│ ├──────────────────────┼───────────────────┼─────────────────────┤ │
│ │ PostgreSQL compat.   │ ✅ 100%            │ ✅ 100%              │ │
│ │ DB engines           │ PG, MySQL,        │ PostgreSQL only     │ │
│ │                      │ SQL Server        │                     │ │
│ │ Transaction speed    │ Standard PG       │ Up to 4x faster     │ │
│ │ Analytical speed     │ Standard PG       │ Up to 100x faster   │ │
│ │ Columnar engine      │ ❌ No              │ ✅ Built-in          │ │
│ │ ML/AI integration    │ ❌ Basic           │ ✅ Vertex AI in SQL  │ │
│ │ Storage engine       │ Standard PG       │ Custom log-based    │ │
│ │ Max storage          │ 64 TB             │ 128 TB              │ │
│ │ Storage scaling      │ ✅ Auto            │ ✅ Auto              │ │
│ │ HA SLA               │ 99.95%            │ 99.99%              │ │
│ │ Failover time        │ ~60-120 seconds   │ < 60 seconds        │ │
│ │ Read replicas        │ ✅ Up to 20        │ ✅ Read pools 1-20   │ │
│ │ Cross-region replica │ ✅ Yes             │ ✅ Yes               │ │
│ │ Public IP            │ ✅ Yes             │ ❌ Private IP only   │ │
│ │ Auth Proxy           │ ✅ Yes             │ ✅ Yes               │ │
│ │ PITR                 │ ✅ Yes (7 days)    │ ✅ Yes (14 days)     │ │
│ │ Maintenance downtime │ Brief restart     │ Near-zero (HA)      │ │
│ │ Query Insights       │ ✅ Built-in        │ ✅ Built-in          │ │
│ │ Serverless option    │ ❌ No              │ ❌ No                │ │
│ │ Free tier            │ ✅ Yes (small)     │ ❌ No                │ │
│ │ Min monthly cost     │ ~$7/month         │ ~$216/month         │ │
│ │ Min HA monthly cost  │ ~$14/month        │ ~$444/month         │ │
│ └──────────────────────┴───────────────────┴─────────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Choose Cloud SQL When...

```
┌─────────────────────────────────────────────────────────────────────┐
│           CHOOSE CLOUD SQL WHEN...                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ✅ Your budget is tight                                              │
│    Cloud SQL starts at ~$7/month. AlloyDB starts at ~$216/month.  │
│    For small apps, that's a 30x cost difference.                   │
│                                                                       │
│ ✅ You need MySQL or SQL Server                                      │
│    AlloyDB only supports PostgreSQL. If your app uses MySQL or    │
│    SQL Server, Cloud SQL is your only managed option on GCP.       │
│                                                                       │
│ ✅ Your workload is small to medium                                  │
│    Personal projects, small business apps, blogs, CMS, simple     │
│    CRUD apps — Cloud SQL handles these perfectly fine.             │
│                                                                       │
│ ✅ You need a public IP endpoint                                     │
│    Cloud SQL supports public IP access (with SSL). AlloyDB is     │
│    private IP only — you'd need Auth Proxy or VPN for external    │
│    access.                                                          │
│                                                                       │
│ ✅ You want a free tier to get started                               │
│    Cloud SQL has a free tier for small instances. AlloyDB does     │
│    not offer any free tier.                                         │
│                                                                       │
│ ✅ You're not sure yet — start with Cloud SQL                       │
│    Since both are 100% PostgreSQL-compatible, you can start with  │
│    Cloud SQL and migrate to AlloyDB later with minimal code        │
│    changes if you outgrow it.                                      │
│                                                                       │
│ Typical Cloud SQL use cases:                                       │
│ ├── Startup MVP or prototype                                       │
│ ├── WordPress / CMS / blog                                         │
│ ├── Small-medium SaaS application                                  │
│ ├── Internal tools and dashboards                                  │
│ ├── Dev/test environments                                          │
│ └── Any app where standard PostgreSQL performance is enough       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Choose AlloyDB When...

```
┌─────────────────────────────────────────────────────────────────────┐
│           CHOOSE ALLOYDB WHEN...                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ✅ You need high-performance transactions                            │
│    E-commerce checkout, financial transactions, high-concurrency   │
│    apps where every millisecond matters. AlloyDB is 4x faster     │
│    for OLTP workloads.                                              │
│                                                                       │
│ ✅ You run both transactions AND analytics on the same database     │
│    AlloyDB's columnar engine gives 100x faster aggregations       │
│    without needing a separate data warehouse. Perfect for apps    │
│    with built-in dashboards and reporting.                         │
│                                                                       │
│ ✅ You're migrating from Oracle or SQL Server (enterprise)          │
│    AlloyDB's performance matches enterprise database expectations │
│    while being PostgreSQL-compatible. Often 50-80% cheaper than   │
│    Oracle licensing.                                                │
│                                                                       │
│ ✅ You need 99.99% availability SLA                                  │
│    Cloud SQL offers 99.95%. If your business requires four-nines  │
│    uptime, AlloyDB is the PostgreSQL option on GCP.                │
│                                                                       │
│ ✅ You want ML/AI integrated with your database                     │
│    AlloyDB's Vertex AI integration lets you call ML models        │
│    directly from SQL queries — embeddings, predictions, and       │
│    vector search without leaving the database.                     │
│                                                                       │
│ ✅ You need to scale reads horizontally                              │
│    Read pools with auto-load-balancing let you add 1-20 nodes    │
│    for read scaling with zero application changes.                 │
│                                                                       │
│ Typical AlloyDB use cases:                                         │
│ ├── Enterprise SaaS with mixed OLTP + analytics                    │
│ ├── Financial services / fintech applications                      │
│ ├── E-commerce with high transaction volumes                       │
│ ├── Oracle/SQL Server migration to PostgreSQL                      │
│ ├── AI/ML-powered applications (vector search, recommendations)   │
│ └── Large-scale applications needing sub-second analytics          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cost Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│           COST COMPARISON — REAL EXAMPLES                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario 1: Small App (startup, personal project)                  │
│ ┌──────────────────────┬───────────────┬─────────────────────────┐ │
│ │ Component            │ Cloud SQL     │ AlloyDB                  │ │
│ ├──────────────────────┼───────────────┼─────────────────────────┤ │
│ │ Instance             │ db-f1-micro   │ 2 vCPU (smallest)       │ │
│ │ Compute cost         │ ~$7/month     │ ~$216/month             │ │
│ │ HA cost              │ ~$14/month    │ ~$444/month             │ │
│ │ Storage (10 GB)      │ ~$2/month     │ ~$2.50/month            │ │
│ │ Total (no HA)        │ ~$9/month     │ ~$218/month             │ │
│ │ Total (with HA)      │ ~$16/month    │ ~$446/month             │ │
│ │                      │               │                         │ │
│ │ ✅ Winner             │ CLOUD SQL ✅   │ Way too expensive for   │ │
│ │                      │ 24x cheaper   │ a small app             │ │
│ └──────────────────────┴───────────────┴─────────────────────────┘ │
│                                                                       │
│ Scenario 2: Medium SaaS (growing startup, ~1000 users)             │
│ ┌──────────────────────┬───────────────┬─────────────────────────┐ │
│ │ Component            │ Cloud SQL     │ AlloyDB                  │ │
│ ├──────────────────────┼───────────────┼─────────────────────────┤ │
│ │ Instance             │ 4 vCPU/26 GB  │ 4 vCPU/32 GB            │ │
│ │ Compute cost (HA)    │ ~$400/month   │ ~$432/month x2 = $864  │ │
│ │ Read replica/pool    │ ~$200/month   │ ~$216/month (1 node)    │ │
│ │ Storage (100 GB)     │ ~$17/month    │ ~$25/month              │ │
│ │ Total                │ ~$617/month   │ ~$1,105/month           │ │
│ │                      │               │                         │ │
│ │ ✅ Winner             │ CLOUD SQL ✅   │ AlloyDB ~2x more, but  │ │
│ │                      │ Still cheaper │ worth it if you need    │ │
│ │                      │               │ analytics + speed       │ │
│ └──────────────────────┴───────────────┴─────────────────────────┘ │
│                                                                       │
│ Scenario 3: Enterprise (10K+ users, mixed OLTP+OLAP)              │
│ ┌──────────────────────┬───────────────┬─────────────────────────┐ │
│ │ Component            │ Cloud SQL     │ AlloyDB                  │ │
│ ├──────────────────────┼───────────────┼─────────────────────────┤ │
│ │ Instance             │ 16 vCPU       │ 16 vCPU                 │ │
│ │ Compute cost (HA)    │ ~$1,600/month │ ~$1,728/month x2=$3,456│ │
│ │ Read replicas/pool   │ ~$800/month x2│ ~$432/month x4 nodes   │ │
│ │ Separate analytics DB│ ~$1,000/month │ ❌ Not needed (columnar)│ │
│ │ Storage (1 TB)       │ ~$170/month   │ ~$245/month             │ │
│ │ Total                │ ~$4,370/month │ ~$5,429/month           │ │
│ │                      │               │                         │ │
│ │ ✅ Winner             │ Close in cost │ ALLOYDB ✅ — no need    │ │
│ │                      │ but needs     │ for separate analytics  │ │
│ │                      │ separate DB   │ DB. Faster. Higher SLA. │ │
│ │                      │ for analytics │ Simpler architecture.   │ │
│ └──────────────────────┴───────────────┴─────────────────────────┘ │
│                                                                       │
│ Rule of thumb:                                                      │
│ ├── Budget < $200/month → Cloud SQL (no question)                 │
│ ├── Budget $200-$1000/month → Cloud SQL (unless you need          │
│ │   columnar engine or 99.99% SLA specifically)                   │
│ ├── Budget > $1000/month + need analytics → AlloyDB starts to     │
│ │   make sense (saves you from maintaining a separate analytics   │
│ │   database)                                                      │
│ └── Migrating from Oracle → AlloyDB (performance expectations     │
│     match, and it's still much cheaper than Oracle licensing)     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 30: Cloud Source Repositories** → `30-cloud-source-repos.md`
