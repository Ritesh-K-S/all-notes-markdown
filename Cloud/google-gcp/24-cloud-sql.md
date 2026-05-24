# Chapter 24: Cloud SQL

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud SQL Fundamentals](#part-1-cloud-sql-fundamentals)
- [Part 2: Database Engines](#part-2-database-engines)
- [Part 3: Creating a Cloud SQL Instance](#part-3-creating-a-cloud-sql-instance)
- [Part 4: Machine Types & Storage](#part-4-machine-types--storage)
- [Part 5: High Availability (HA)](#part-5-high-availability-ha)
- [Part 6: Read Replicas](#part-6-read-replicas)
- [Part 7: Backups & Point-in-Time Recovery](#part-7-backups--point-in-time-recovery)
- [Part 8: Connectivity & Networking](#part-8-connectivity--networking)
- [Part 9: Security](#part-9-security)
- [Part 10: Maintenance & Updates](#part-10-maintenance--updates)
- [Part 11: Import & Export](#part-11-import--export)
- [Part 12: Database Flags & Configuration](#part-12-database-flags--configuration)
- [Part 13: Monitoring & Performance](#part-13-monitoring--performance)
- [Part 14: Terraform & CLI](#part-14-terraform--cli)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Quick Reference](#quick-reference)
- [Console Walkthrough: Managing & Deleting Instances](#console-walkthrough-managing--deleting-instances)
- [Connection Pooling](#connection-pooling)
- [Query Insights](#query-insights)
- [Additional gcloud Commands](#additional-gcloud-commands)
- [What's Next?](#whats-next)

---

## Overview

Cloud SQL is Google Cloud's fully managed relational database service supporting MySQL, PostgreSQL, and SQL Server. It handles provisioning, patching, backups, replication, and failover — you focus on your schema and queries.

```
What you'll learn:
├── Cloud SQL Fundamentals
│   ├── Managed relational database
│   ├── MySQL, PostgreSQL, SQL Server
│   ├── When to use Cloud SQL vs alternatives
│   └── Pricing model
├── Instance Configuration
│   ├── Machine types (shared-core to 96 vCPUs)
│   ├── Storage (SSD vs HDD, auto-resize)
│   ├── Database flags
│   └── Maintenance windows
├── High Availability
│   ├── Regional HA (automatic failover)
│   ├── Failover mechanics
│   └── HA vs Read Replica
├── Read Replicas
│   ├── Same-region replicas
│   ├── Cross-region replicas
│   ├── Cascading replicas
│   └── External replicas
├── Backups & Recovery
│   ├── Automated backups
│   ├── On-demand backups
│   ├── Point-in-time recovery (PITR)
│   └── Cross-region backup
├── Connectivity
│   ├── Public IP (with authorized networks)
│   ├── Private IP (VPC)
│   ├── Cloud SQL Auth Proxy
│   └── Cloud SQL Connectors
├── Security
│   ├── Encryption (at rest, in transit)
│   ├── IAM database authentication
│   ├── SSL/TLS enforcement
│   └── CMEK
├── Monitoring & Performance
├── Terraform & CLI
└── Real-world patterns
```

---

## Part 1: Cloud SQL Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS CLOUD SQL?                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud SQL = Fully managed relational database service.             │
│                                                                       │
│ "Fully managed" means Google handles:                               │
│ ├── Provisioning (hardware, OS, database software)               │
│ ├── Patching (OS and database engine updates)                    │
│ ├── Backups (automated daily + transaction logs)                │
│ ├── Replication (HA failover, read replicas)                    │
│ ├── Monitoring (built-in metrics and insights)                  │
│ ├── Storage scaling (auto-resize)                                │
│ └── Failover (automatic with HA configuration)                  │
│                                                                       │
│ You manage:                                                         │
│ ├── Schema design (tables, indexes, views)                      │
│ ├── Queries (optimization, slow query analysis)                 │
│ ├── Users & permissions (database-level grants)                 │
│ ├── Application connection logic                                │
│ ├── Database flags / tuning parameters                          │
│ └── Choosing the right instance size                            │
│                                                                       │
│ ┌─────────────────────────────────────────────────────┐            │
│ │  Your Application                                    │            │
│ │  (App Engine / Cloud Run / GKE / VM / External)     │            │
│ └────────────────┬────────────────────────────────────┘            │
│                  │ SQL connection                                   │
│                  ▼                                                  │
│ ┌─────────────────────────────────────────────────────┐            │
│ │  Cloud SQL Instance                                  │            │
│ │  ┌──────────────────────────────────────────────┐   │            │
│ │  │ Database Engine (MySQL / PostgreSQL / SQL Svr)│   │            │
│ │  ├──────────────────────────────────────────────┤   │            │
│ │  │ Compute (vCPUs + RAM)                        │   │            │
│ │  ├──────────────────────────────────────────────┤   │            │
│ │  │ Storage (SSD / HDD, auto-resize)             │   │            │
│ │  ├──────────────────────────────────────────────┤   │            │
│ │  │ Automated Backups + Binary/WAL Logs          │   │            │
│ │  └──────────────────────────────────────────────┘   │            │
│ └─────────────────────────────────────────────────────┘            │
│                                                                       │
│ NOT a "database as a service" like Firestore — Cloud SQL gives   │
│ you a traditional SQL database with standard SQL access.         │
│                                                                       │
│ AWS equivalent: Amazon RDS                                         │
│ Azure equivalent: Azure SQL Database / Azure Database for MySQL  │
│                   / Azure Database for PostgreSQL                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### When to Use Cloud SQL vs Alternatives

```
┌──────────────────────┬──────────────────────────────────────────────┐
│ Service              │ When to use                                  │
├──────────────────────┼──────────────────────────────────────────────┤
│ Cloud SQL            │ Traditional relational workloads, OLTP,     │
│                      │ ≤ 64 TB, MySQL/PostgreSQL/SQL Server        │
│                      │ compatible, need managed service             │
│                      │                                              │
│ AlloyDB              │ PostgreSQL-compatible, need higher perf      │
│                      │ than Cloud SQL, analytics + OLTP hybrid,    │
│                      │ columnar engine, larger scale                │
│                      │                                              │
│ Cloud Spanner        │ Global scale, unlimited horizontal scaling, │
│                      │ strong consistency, 99.999% availability,   │
│                      │ multi-region, very expensive                 │
│                      │                                              │
│ Firestore            │ Document database, NoSQL, serverless,       │
│                      │ mobile/web apps, real-time sync              │
│                      │                                              │
│ Bigtable             │ Wide-column NoSQL, petabyte-scale, IoT,     │
│                      │ time series, analytics, low-latency          │
│                      │                                              │
│ Self-managed on GCE  │ Need OS access, custom extensions, exotic   │
│                      │ configs, database not supported by Cloud SQL│
└──────────────────────┴──────────────────────────────────────────────┘

⚡ Cloud SQL is right for 90%+ of traditional database needs.
  Use Spanner only if you need global scale or 99.999% SLA.
  Use AlloyDB if you need PostgreSQL but outgrow Cloud SQL.
```

---

## Part 2: Database Engines

```
┌─────────────────────────────────────────────────────────────────────┐
│           SUPPORTED DATABASE ENGINES                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬────────────────────────────────────────────┐  │
│ │ Engine           │ Details                                     │  │
│ ├──────────────────┼────────────────────────────────────────────┤  │
│ │ MySQL            │ Versions: 5.7, 8.0, 8.4                   │  │
│ │                  │ Most popular, widest tooling support       │  │
│ │                  │ Max 64 TB storage                           │  │
│ │                  │ Max 96 vCPUs, 624 GB RAM                   │  │
│ │                  │ Community edition (InnoDB engine)           │  │
│ │                  │                                             │  │
│ │ PostgreSQL       │ Versions: 13, 14, 15, 16, 17              │  │
│ │                  │ Advanced features (JSONB, CTEs, extensions)│  │
│ │                  │ Max 64 TB storage                           │  │
│ │                  │ Max 96 vCPUs, 624 GB RAM                   │  │
│ │                  │ Extensions: PostGIS, pgvector, pg_cron,    │  │
│ │                  │   pgAudit, and many more                   │  │
│ │                  │ ⚡ Best choice for most new projects        │  │
│ │                  │                                             │  │
│ │ SQL Server       │ Versions: 2017, 2019, 2022                │  │
│ │                  │ Editions: Express, Web, Standard, Ent.    │  │
│ │                  │ Max 64 TB storage                           │  │
│ │                  │ Max 96 vCPUs, 624 GB RAM                   │  │
│ │                  │ Windows + Linux support                    │  │
│ │                  │ License included in pricing                │  │
│ │                  │ ⚡ Most expensive (SQL Server licensing)    │  │
│ └──────────────────┴────────────────────────────────────────────┘  │
│                                                                       │
│ Engine choice considerations:                                       │
│ ├── Existing application: Match current database engine         │
│ ├── New project: PostgreSQL recommended (most features)        │
│ ├── WordPress/PHP ecosystem: MySQL                              │
│ ├── .NET/Microsoft ecosystem: SQL Server                       │
│ ├── Need PostGIS (geospatial): PostgreSQL                      │
│ ├── Need pgvector (AI/ML embeddings): PostgreSQL               │
│ └── Budget-conscious: MySQL or PostgreSQL (no license cost)    │
│                                                                       │
│ ⚡ You CANNOT change the engine after creation!                    │
│   MySQL instance stays MySQL forever.                             │
│   To switch: Export data → create new instance → import.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating a Cloud SQL Instance

```
Console → SQL → Create Instance → Choose database engine

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE CLOUD SQL INSTANCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Choose a database engine:                                           │
│   ○ MySQL    ○ PostgreSQL    ○ SQL Server                          │
│                                                                       │
│ === Instance Info ===                                               │
│                                                                       │
│ Instance ID: [webapp-db-prod]                                      │
│ ⚡ Cannot change after creation! Choose carefully.                  │
│ ⚡ Must be unique within the project.                               │
│ ⚡ If you delete an instance, its name cannot be reused for        │
│   ~1 week (to prevent DNS conflicts).                             │
│                                                                       │
│ Password (root/postgres/sqlserver user):                           │
│ [••••••••••••]                                                     │
│ ⚡ MySQL: root user                                                 │
│ ⚡ PostgreSQL: postgres user                                        │
│ ⚡ SQL Server: sqlserver user                                       │
│                                                                       │
│ Database version: [PostgreSQL 16]                                  │
│ ⚡ Can do MINOR version upgrades (16.1 → 16.2) automatically.     │
│ ⚡ MAJOR version upgrades (15 → 16) require in-place upgrade      │
│   or blue-green process.                                          │
│                                                                       │
│ Cloud SQL edition:                                                  │
│   ○ Enterprise (standard — recommended for prod) ✅               │
│   ○ Enterprise Plus (higher perf, data cache, advanced HA)       │
│                                                                       │
│ Preset:                                                             │
│   ○ Development (small, no HA)                                    │
│   ○ Sandbox (tiny, for testing)                                   │
│   ○ Production (HA enabled, SSD, larger machine) ✅               │
│                                                                       │
│ Region: [asia-south1]                                              │
│ ⚡ Cannot change after creation!                                    │
│ ⚡ Choose closest to your application servers.                      │
│                                                                       │
│ Zonal availability:                                                 │
│   ○ Single zone (cheaper, no automatic failover)                  │
│   ○ Multiple zones (HA) ✅ → primary zone + standby zone         │
│     Primary zone: [asia-south1-a]                                 │
│     Secondary zone: [asia-south1-b] or [Any]                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Machine Types & Storage

```
┌─────────────────────────────────────────────────────────────────────┐
│           MACHINE TYPES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud SQL uses Compute Engine VMs under the hood (managed by       │
│ Google — you don't see them). You choose the machine shape.       │
│                                                                       │
│ Categories:                                                         │
│                                                                       │
│ SHARED-CORE (burstable, cheapest):                                 │
│ ├── db-f1-micro: 0.6 GB RAM, shared vCPU                        │
│ │   ⚡ Free tier eligible! Great for dev/test                     │
│ ├── db-g1-small: 1.7 GB RAM, shared vCPU                        │
│ └── ⚠️ NOT for production — shared CPU, throttled under load     │
│                                                                       │
│ STANDARD (dedicated vCPUs):                                        │
│ ├── db-custom-N-M: N vCPUs, M MB RAM                            │
│ │   Examples:                                                     │
│ │   ├── db-custom-1-3840:   1 vCPU, 3.75 GB                    │
│ │   ├── db-custom-2-7680:   2 vCPUs, 7.5 GB                    │
│ │   ├── db-custom-4-15360:  4 vCPUs, 15 GB                     │
│ │   ├── db-custom-8-30720:  8 vCPUs, 30 GB                     │
│ │   ├── db-custom-16-61440: 16 vCPUs, 60 GB                    │
│ │   ├── db-custom-32-122880: 32 vCPUs, 120 GB                  │
│ │   └── db-custom-96-638976: 96 vCPUs, 624 GB (max)           │
│ │                                                                 │
│ ├── You can customize vCPU:RAM ratio                             │
│ │   ├── Min RAM: 3.75 GB per vCPU                                │
│ │   ├── Max RAM: 6.5 GB per vCPU                                 │
│ │   └── RAM must be in 256 MB increments                        │
│ │                                                                 │
│ └── ⚡ Right-sizing: Start with estimate, monitor, resize        │
│     Can change machine type with ~1-2 min downtime (restart).   │
│                                                                       │
│ HIGH-MEMORY (Enterprise Plus):                                     │
│ ├── Up to 96 vCPUs, 624 GB RAM                                  │
│ ├── Data cache (local SSD for hot data — faster reads)          │
│ └── Advanced HA (faster failover)                               │
│                                                                       │
│ ⚡ You CAN change machine type after creation!                     │
│   But it requires a restart (~1-2 minutes downtime).             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           STORAGE                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Storage type:                                                       │
│   ○ SSD (recommended ✅): Lower latency, higher IOPS             │
│   ○ HDD: Cheaper, higher latency, good for large read-heavy     │
│     workloads where IOPS doesn't matter                          │
│                                                                       │
│ Storage capacity:                                                   │
│ ├── Min: 10 GB                                                   │
│ ├── Max: 64 TB                                                   │
│ └── IOPS scales with storage size (like Persistent Disk)        │
│                                                                       │
│ Auto storage increase:                                              │
│ ├── ☑ Enable automatic storage increases ✅                      │
│ ├── When storage usage hits threshold (~85-90%)                  │
│ │   Google automatically adds more capacity                     │
│ ├── Increments: Adds 25% of current size or min needed          │
│ ├── Max limit: You can set a max (e.g., 500 GB)                │
│ │   ⚡ Set a max to prevent runaway costs from data bloat!       │
│ ├── ⚠️ Storage auto-increase CANNOT be reversed!                 │
│ │   Once increased, you can't shrink storage.                   │
│ └── ⚠️ Storage auto-increase can cause brief performance impact │
│                                                                       │
│ ⚡ STORAGE CANNOT BE DECREASED — EVER!                             │
│   Start small, let auto-increase handle growth.                  │
│   If you over-provision, you pay for unused space forever.       │
│                                                                       │
│ AWS equivalent:                                                     │
│ ├── RDS SSD → gp3/io1                                            │
│ ├── RDS storage autoscaling → Auto storage increase             │
│ └── Same limitation: Cannot shrink storage                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: High Availability (HA)

```
┌─────────────────────────────────────────────────────────────────────┐
│           HIGH AVAILABILITY                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud SQL HA = Regional instance with automatic failover.          │
│                                                                       │
│ How it works:                                                       │
│                                                                       │
│ ┌───────────────┐     ┌───────────────┐                            │
│ │  Zone A        │     │  Zone B        │                            │
│ │ ┌───────────┐ │     │ ┌───────────┐ │                            │
│ │ │ PRIMARY   │ │─────│►│ STANDBY   │ │                            │
│ │ │ Instance  │ │sync │ │ Instance  │ │                            │
│ │ │ (active)  │ │repl │ │ (passive) │ │                            │
│ │ └───────────┘ │     │ └───────────┘ │                            │
│ │ ┌───────────┐ │     │ ┌───────────┐ │                            │
│ │ │ Disk      │ │─────│►│ Disk      │ │                            │
│ │ │ (primary) │ │sync │ │ (replica) │ │                            │
│ │ └───────────┘ │     │ └───────────┘ │                            │
│ └───────────────┘     └───────────────┘                            │
│                                                                       │
│ Key mechanics:                                                      │
│ ├── SYNCHRONOUS replication from primary to standby             │
│ │   (every write confirmed on both before ACK to client)        │
│ ├── Standby is in a DIFFERENT ZONE (same region)               │
│ ├── Standby does NOT serve read traffic (it's passive)         │
│ │   ⚡ Unlike Read Replicas — HA standby is for failover ONLY   │
│ ├── Same IP address after failover (DNS-based, ~seconds)       │
│ ├── Automatic failover triggers:                                │
│ │   ├── Zone failure                                            │
│ │   ├── Primary instance unresponsive                          │
│ │   ├── Primary VM crash                                       │
│ │   └── OS/instance maintenance                                │
│ ├── Failover time: ~60 seconds typically                       │
│ │   (Enterprise Plus: ~10 seconds with faster failover)        │
│ └── Manual failover: You can trigger for testing               │
│                                                                       │
│ HA costs:                                                           │
│ ├── 2x compute cost (primary + standby)                         │
│ ├── 2x storage cost (data replicated to both zones)            │
│ └── ~2x total cost compared to single-zone                     │
│                                                                       │
│ HA vs Read Replica:                                                 │
│ ┌──────────────────┬───────────────────┬──────────────────────┐   │
│ │                  │ HA (Standby)      │ Read Replica         │   │
│ ├──────────────────┼───────────────────┼──────────────────────┤   │
│ │ Purpose          │ Automatic failover│ Scale reads          │   │
│ │ Serves traffic   │ ❌ (passive)      │ ✅ Read queries      │   │
│ │ Replication      │ Synchronous       │ Asynchronous         │   │
│ │ Data lag         │ 0 (sync)          │ Seconds to minutes   │   │
│ │ Zones            │ Different zone    │ Same or diff zone    │   │
│ │ Regions          │ Same region       │ Same or diff region  │   │
│ │ Failover         │ Automatic         │ Manual promotion     │   │
│ │ IP address       │ Same after fail   │ Different IP         │   │
│ └──────────────────┴───────────────────┴──────────────────────┘   │
│                                                                       │
│ ⚡ Use BOTH for production: HA for failover + Read Replicas       │
│   for scaling reads. They solve different problems.              │
│                                                                       │
│ AWS equivalent: RDS Multi-AZ deployment                            │
│ Azure equivalent: Zone-redundant deployment                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Read Replicas

```
┌─────────────────────────────────────────────────────────────────────┐
│           READ REPLICAS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Copies of the primary instance that serve READ queries.     │
│       Offload read traffic from the primary.                      │
│                                                                       │
│ ┌──────────┐    async    ┌──────────────┐                          │
│ │ PRIMARY  │────repl────►│ Read Replica │  ← serves SELECT only  │
│ │ Instance │             │ (same region)│                          │
│ └──────────┘             └──────────────┘                          │
│      │                                                              │
│      │         async    ┌──────────────────┐                       │
│      └────────repl─────►│ Cross-Region     │  ← DR + local reads │
│                          │ Read Replica     │                       │
│                          │ (different region)│                      │
│                          └──────────────────┘                       │
│                                                                       │
│ Key properties:                                                     │
│ ├── ASYNCHRONOUS replication (slight lag behind primary)        │
│ │   Lag typically: milliseconds to seconds                      │
│ │   Under heavy write load: can be seconds to minutes           │
│ │                                                                 │
│ ├── READ-ONLY: Only SELECT queries work                         │
│ │   INSERT/UPDATE/DELETE → must go to primary                  │
│ │                                                                 │
│ ├── SAME or DIFFERENT region:                                    │
│ │   ├── Same region: Lower latency, scale reads                │
│ │   └── Cross-region: DR, users in other geos                  │
│ │                                                                 │
│ ├── Different IP address from primary                           │
│ │   Application must know replica IP and route reads            │
│ │                                                                 │
│ ├── Can have up to 10 read replicas per primary                │
│ │                                                                 │
│ ├── CASCADING replicas:                                          │
│ │   Primary → Replica A → Replica B                            │
│ │   Reduces load on primary for replication                    │
│ │   Replica B can be in a third region                         │
│ │                                                                 │
│ ├── PROMOTION: A read replica can be promoted to standalone    │
│ │   ├── Breaks replication permanently                         │
│ │   ├── Becomes an independent read-write instance             │
│ │   ├── Use for: DR failover, region migration                 │
│ │   └── ⚠️ Cannot be reversed! (no "demote back to replica")   │
│ │                                                                 │
│ └── Replica has its OWN machine type and storage               │
│     Can be different size than primary                          │
│     (but should be >= primary for performance)                  │
│                                                                       │
│ Limitations:                                                        │
│ ├── Cannot have read replicas of read replicas (except cascade)│
│ ├── Replica cannot have HA (no standby for replica)            │
│ ├── Replica does not have automated backups                    │
│ └── Some operations on primary briefly pause replication       │
│                                                                       │
│ AWS equivalent: RDS Read Replicas (same concept)                  │
│ Azure equivalent: Read replicas (Azure DB for MySQL/PostgreSQL)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Region Read Replica for DR

```
┌─────────────────────────────────────────────────────────────────────┐
│           CROSS-REGION REPLICA FOR DR                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern: Use cross-region read replica as a DR strategy.           │
│                                                                       │
│   asia-south1 (primary)         us-central1 (DR)                  │
│   ┌──────────────────┐          ┌──────────────────┐               │
│   │ Primary Instance │──async──►│ Cross-Region     │               │
│   │ (read + write)   │  repl    │ Read Replica     │               │
│   │ HA enabled       │          │ (read only)      │               │
│   └──────────────────┘          └──────────────────┘               │
│                                                                       │
│ Normal: Reads from primary (or same-region replica)               │
│ Disaster: Promote cross-region replica → becomes new primary     │
│                                                                       │
│ RPO (Recovery Point Objective): Seconds to minutes               │
│   (depends on replication lag at time of failure)                │
│ RTO (Recovery Time Objective): ~Minutes                           │
│   (time to detect failure + promote replica + update app config) │
│                                                                       │
│ ⚡ Cross-region replication incurs network egress charges.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Backups & Point-in-Time Recovery

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKUPS                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two backup types:                                                   │
│                                                                       │
│ 1. AUTOMATED BACKUPS:                                               │
│    ├── Run daily during your chosen backup window                │
│    ├── 4-hour backup window (e.g., 02:00–06:00)                 │
│    ├── Retention: 1–365 days (default: 7 days)                  │
│    ├── Stored in the same region as the instance                │
│    │   (or multi-region for cross-region backup)                │
│    ├── Incremental after first full backup                      │
│    └── Enabled by default on new instances ✅                    │
│                                                                       │
│ 2. ON-DEMAND BACKUPS:                                               │
│    ├── Manually triggered anytime                                │
│    ├── Retained until you explicitly delete them                │
│    ├── ⚡ NOT automatically cleaned up!                            │
│    ├── Use before: schema migrations, major changes             │
│    └── Count toward backup storage costs                        │
│                                                                       │
│ Backup storage location:                                            │
│ ├── Default: Same region as instance                             │
│ ├── Multi-region: Stored in a multi-region (e.g., "us", "eu")  │
│ │   ⚡ Cross-region backup = protects against regional disaster │
│ ├── Custom: Specific region different from instance              │
│ └── Multi-region costs more (cross-region storage + transfer)   │
│                                                                       │
│ Console → SQL → Instance → Backups:                                │
│   Automated backup: Enabled                                       │
│   Backup window: 02:00 – 06:00                                   │
│   Retention: [7] days                                              │
│   Location: ○ Same as instance  ○ Multi-region  ○ Custom region  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Point-in-Time Recovery (PITR)

```
┌─────────────────────────────────────────────────────────────────────┐
│           POINT-IN-TIME RECOVERY (PITR)                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Restore your database to any point in time within the       │
│       backup retention window. Not just to a backup — to any     │
│       second.                                                     │
│                                                                       │
│ How it works:                                                       │
│ 1. Google continuously captures binary logs (MySQL) or           │
│    WAL (Write-Ahead Logs, PostgreSQL) or transaction logs        │
│    (SQL Server)                                                   │
│ 2. These logs record every write operation                       │
│ 3. To restore: Replay backup + logs up to the requested time    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Timeline:                                                     │  │
│ │                                                               │  │
│ │ ──[Backup 1]────────[Backup 2]────────[Backup 3]───[NOW]── │  │
│ │       │                                    │                  │  │
│ │       │     ┌─── Transaction/WAL Logs ───┐│                  │  │
│ │       │     │  Every write is logged      ││                  │  │
│ │       │     └─────────────────────────────┘│                  │  │
│ │       │                    ▲                │                  │  │
│ │       │                    │                │                  │  │
│ │       │              PITR target           │                  │  │
│ │       │         (any second here!)          │                  │  │
│ │       │                                     │                  │  │
│ │ Restore = Backup 2 + replay logs to target                   │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Use cases:                                                          │
│ ├── Accidental DELETE/DROP — restore to 1 second before!        │
│ ├── Bad migration — roll back to pre-migration state           │
│ └── Data corruption — find last clean point, restore           │
│                                                                       │
│ ⚡ PITR creates a NEW instance with data at that point in time.   │
│   It does NOT restore in-place.                                   │
│   You get a new instance → update app config → delete old.      │
│                                                                       │
│ ⚡ PITR must be enabled (it's on by default for new instances).   │
│   Binary/WAL log retention: 1–7 days (default: 7 days).         │
│   Logs use storage space on the instance.                        │
│                                                                       │
│ Console → SQL → Instance → Backups → Restore                     │
│   ○ From a backup (specific backup)                              │
│   ○ From a point in time (PITR — enter timestamp)               │
│                                                                       │
│ CLI:                                                                │
│ gcloud sql instances clone SOURCE_INSTANCE NEW_INSTANCE \        │
│   --point-in-time="2026-05-17T10:30:00.000Z"                    │
│                                                                       │
│ AWS equivalent: RDS Point-in-Time Recovery (same concept)        │
│ Azure equivalent: Point-in-time restore                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Connectivity & Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECTION METHODS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud SQL supports 4 connection methods:                            │
│                                                                       │
│ 1. PUBLIC IP + Authorized Networks                                 │
│ 2. PRIVATE IP (VPC)                                                │
│ 3. Cloud SQL Auth Proxy                                            │
│ 4. Cloud SQL Connectors (language libraries)                      │
│                                                                       │
│ ⚡ Best practice for production:                                    │
│   Private IP + Cloud SQL Auth Proxy OR Connectors                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Method 1: Public IP

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUBLIC IP + AUTHORIZED NETWORKS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance gets a public IP address (e.g., 34.93.x.x).             │
│ You whitelist specific IP ranges that can connect.                │
│                                                                       │
│ Console → SQL → Instance → Connections → Networking:              │
│   ☑ Public IP                                                      │
│   Authorized networks:                                             │
│     Name: [office]  Network: [203.0.113.0/24]                    │
│     Name: [vpn]     Network: [10.0.0.0/8]                        │
│     ⚡ Only these IPs can reach the database!                      │
│                                                                       │
│ Pros: Simple, works from anywhere (office, home, CI/CD)          │
│ Cons:                                                               │
│ ├── Public IP exposed to internet (even with whitelist)          │
│ ├── Must manage IP whitelist                                     │
│ ├── Dynamic IPs (home/office) are problematic                   │
│ └── Not recommended for production ⚠️                             │
│                                                                       │
│ ⚡ For quick testing only. Use Private IP for production.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Method 2: Private IP

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIVATE IP (VPC PEERING)                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance gets a private IP in your VPC (e.g., 10.0.1.5).         │
│ Only accessible from within your VPC.                             │
│                                                                       │
│ How it works:                                                       │
│ ├── Google creates a VPC Peering between your VPC and the       │
│ │   Cloud SQL VPC (managed by Google)                            │
│ ├── Private Services Access (PSA) allocates an IP range         │
│ ├── Instance gets a private IP from that range                  │
│ └── Your VMs/GKE connect via private IP (no internet)           │
│                                                                       │
│ ┌──────────────────────┐    VPC     ┌────────────────────┐       │
│ │  Your VPC            │   Peering  │  Google-managed VPC │       │
│ │  ┌──────┐            │◄──────────►│  ┌──────────────┐  │       │
│ │  │ VM   │───10.0.1.5─┤            │  │ Cloud SQL    │  │       │
│ │  └──────┘            │            │  │ 10.0.1.5     │  │       │
│ │  ┌──────┐            │            │  └──────────────┘  │       │
│ │  │ GKE  │───10.0.1.5─┤            │                    │       │
│ │  └──────┘            │            │                    │       │
│ └──────────────────────┘            └────────────────────┘       │
│                                                                       │
│ Setup:                                                              │
│ 1. Enable Private Services Access on your VPC:                   │
│    Console → VPC → Private Service Connection                    │
│    Allocate an IP range (e.g., 10.100.0.0/16)                   │
│ 2. Create Cloud SQL instance with Private IP enabled             │
│ 3. Connect from VMs/GKE using the private IP                    │
│                                                                       │
│ Pros:                                                               │
│ ├── No public internet exposure ✅                                │
│ ├── Lower latency (private network)                              │
│ ├── No IP whitelisting needed                                   │
│ └── Recommended for production ✅                                 │
│                                                                       │
│ Cons:                                                               │
│ ├── Cannot connect from outside VPC (need VPN/Interconnect)    │
│ ├── PSA setup required (one-time per VPC)                       │
│ └── IP allocation planning needed                               │
│                                                                       │
│ ⚡ Private IP CANNOT be added after creation for some configs.    │
│   Enable it during instance creation!                            │
│                                                                       │
│ AWS equivalent: RDS in VPC (private subnets)                      │
│ Azure equivalent: Private Endpoint / VNet Integration             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Method 3: Cloud SQL Auth Proxy

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD SQL AUTH PROXY                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A small binary that runs alongside your app and creates     │
│       an authenticated, encrypted tunnel to Cloud SQL.            │
│       Your app connects to localhost, the proxy handles the rest.│
│                                                                       │
│ ┌─────────────────────────────────────┐                            │
│ │  Your VM / Container / GKE Pod       │                            │
│ │                                       │                            │
│ │  ┌─────────┐    ┌────────────────┐  │                            │
│ │  │ Your App│───►│ Auth Proxy     │  │                            │
│ │  │ connect │    │ (localhost:5432)│  │                            │
│ │  │ to      │    └───────┬────────┘  │                            │
│ │  │ localhost│           │            │                            │
│ │  └─────────┘           │ Encrypted  │                            │
│ │                        │ tunnel     │                            │
│ └────────────────────────┼────────────┘                            │
│                          ▼                                          │
│              ┌─────────────────────┐                               │
│              │  Cloud SQL Instance  │                               │
│              └─────────────────────┘                               │
│                                                                       │
│ Benefits:                                                           │
│ ├── ENCRYPTED connection (TLS) without managing certificates    │
│ ├── IAM-BASED auth (uses service account, no password in config)│
│ ├── No IP whitelisting needed                                   │
│ ├── Works with both public and private IP                       │
│ ├── Automatic credential rotation                               │
│ └── ⚡ Recommended connection method for most scenarios ✅        │
│                                                                       │
│ Running the proxy:                                                  │
│ # Download                                                         │
│ curl -o cloud-sql-proxy \                                         │
│   https://storage.googleapis.com/cloud-sql-connectors/            │
│   cloud-sql-proxy/v2.x.x/cloud-sql-proxy.linux.amd64             │
│ chmod +x cloud-sql-proxy                                          │
│                                                                       │
│ # Run (TCP mode — app connects to localhost:5432)                 │
│ ./cloud-sql-proxy \                                                │
│   PROJECT:REGION:INSTANCE_NAME \                                  │
│   --port=5432                                                     │
│                                                                       │
│ # Your app connects to:                                           │
│ host=127.0.0.1 port=5432 dbname=mydb user=myuser password=...   │
│                                                                       │
│ In GKE: Run as a sidecar container in the same pod.              │
│ In Cloud Run: Use built-in Cloud SQL connection (automatic).     │
│ In App Engine: Built-in support (just set connection string).    │
│                                                                       │
│ AWS equivalent: No direct equivalent (RDS Proxy is different —   │
│   it's a connection pooler, not an auth tunnel)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Method 4: Cloud SQL Connectors

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD SQL CONNECTORS (LANGUAGE LIBRARIES)                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Language-specific libraries that do what the Auth Proxy     │
│       does, but embedded IN your application code. No sidecar.   │
│                                                                       │
│ Available for:                                                      │
│ ├── Python: cloud-sql-python-connector                           │
│ ├── Java: cloud-sql-jdbc-socket-factory                          │
│ ├── Node.js: cloud-sql-nodejs-connector                          │
│ ├── Go: cloud-sql-go-connector                                   │
│ └── .NET: Google.Cloud.CloudSql.Connector                        │
│                                                                       │
│ Python example:                                                     │
│ from google.cloud.sql.connector import Connector                  │
│ import sqlalchemy                                                  │
│                                                                       │
│ connector = Connector()                                            │
│                                                                       │
│ def getconn():                                                     │
│     conn = connector.connect(                                     │
│         "project:region:instance",                                │
│         "pg8000",           # or pymysql for MySQL               │
│         user="myuser",                                            │
│         password="mypass",                                        │
│         db="mydb",                                                │
│     )                                                              │
│     return conn                                                    │
│                                                                       │
│ engine = sqlalchemy.create_engine(                                │
│     "postgresql+pg8000://",                                       │
│     creator=getconn,                                              │
│ )                                                                   │
│                                                                       │
│ ⚡ No separate proxy process needed — the connector is in-process.│
│ ⚡ Handles TLS, IAM auth, automatic cert rotation.                 │
│ ⚡ Best for serverless (Cloud Run, Cloud Functions) where you     │
│   can't easily run a sidecar.                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Connection Method Decision Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHICH CONNECTION METHOD TO USE?                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────────────────┬───────────────────────────────────┐ │
│ │ Scenario                   │ Recommended method                │ │
│ ├────────────────────────────┼───────────────────────────────────┤ │
│ │ VM in same VPC             │ Private IP + Auth Proxy           │ │
│ │ GKE pods                   │ Private IP + Auth Proxy (sidecar)│ │
│ │ Cloud Run                  │ Built-in connector OR Connector  │ │
│ │                            │ library (no proxy needed)         │ │
│ │ Cloud Functions            │ Connector library                 │ │
│ │ App Engine                 │ Built-in Unix socket              │ │
│ │ On-premises (via VPN)      │ Private IP (direct)              │ │
│ │ Developer laptop           │ Auth Proxy (local) + Public IP   │ │
│ │ CI/CD (GitHub Actions)     │ Auth Proxy + Public IP            │ │
│ │ External SaaS              │ Public IP + SSL + whitelist       │ │
│ └────────────────────────────┴───────────────────────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY FEATURES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. ENCRYPTION AT REST:                                              │
│    ├── All data encrypted by default (Google-managed keys)       │
│    ├── CMEK available (Cloud KMS customer-managed keys)          │
│    └── CSEK NOT supported for Cloud SQL                          │
│                                                                       │
│ 2. ENCRYPTION IN TRANSIT:                                          │
│    ├── SSL/TLS encryption for client connections                 │
│    ├── Server CA certificate provided by Google                  │
│    ├── Can enforce SSL-only connections (reject non-SSL)        │
│    │   Console → Instance → Connections → Security              │
│    │   ☑ Allow only SSL connections                              │
│    └── Auth Proxy / Connectors handle TLS automatically         │
│                                                                       │
│ 3. IAM DATABASE AUTHENTICATION:                                    │
│    ├── Login with IAM user/service account (no password!)       │
│    ├── Short-lived OAuth2 token instead of static password     │
│    ├── Centralized access control via IAM                       │
│    ├── Audit via Cloud Audit Logs                               │
│    ├── MySQL: Supported ✅                                       │
│    ├── PostgreSQL: Supported ✅                                   │
│    └── SQL Server: NOT supported ❌                              │
│                                                                       │
│    Setup:                                                          │
│    a. Enable IAM authentication on instance:                     │
│       Console → Instance → Users → Add User → IAM              │
│    b. Grant database role: cloudsql.instanceUser                │
│    c. Connect: Auth Proxy + --auto-iam-authn flag              │
│       or Connector library with enable_iam_auth=True           │
│                                                                       │
│ 4. DATABASE USERS & PERMISSIONS:                                   │
│    ├── Built-in database users (standard SQL GRANT/REVOKE)     │
│    ├── Default user: root (MySQL) / postgres (PG) / sqlserver  │
│    ├── Create app-specific users with minimal permissions      │
│    └── ⚡ Never use root/postgres user in application code!      │
│                                                                       │
│ 5. VPC SERVICE CONTROLS:                                           │
│    ├── Create a security perimeter around Cloud SQL             │
│    ├── Prevents data exfiltration even if credentials leak     │
│    └── Enterprise security requirement                          │
│                                                                       │
│ 6. PASSWORD POLICY (PostgreSQL/MySQL):                             │
│    ├── Min length                                                │
│    ├── Complexity requirements                                   │
│    ├── Password expiration                                       │
│    └── Disallow username in password                            │
│                                                                       │
│ AWS: RDS IAM auth, SSL/TLS, KMS encryption — same concepts     │
│ Azure: Azure AD auth, TLS, customer-managed keys — same concepts│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Maintenance & Updates

```
┌─────────────────────────────────────────────────────────────────────┐
│           MAINTENANCE                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Google periodically updates Cloud SQL instances for:               │
│ ├── Security patches (OS and database engine)                    │
│ ├── Minor version upgrades                                       │
│ ├── Infrastructure improvements                                  │
│ └── Bug fixes                                                    │
│                                                                       │
│ Maintenance window:                                                 │
│ ├── YOU choose when maintenance can happen                      │
│ │   Day of week: [Saturday]                                     │
│ │   Hour: [02:00]                                                │
│ ├── Maintenance causes brief downtime (~1-5 minutes)           │
│ ├── With HA: Failover to standby, then update standby          │
│ │   (reduces downtime to seconds)                               │
│ └── Can deny maintenance for up to 90 days                     │
│                                                                       │
│ Deny maintenance period:                                            │
│ ├── Block maintenance during critical business periods          │
│ │   (e.g., Black Friday, year-end processing)                  │
│ ├── Max: 90 days                                                 │
│ └── After 90 days, maintenance is forced                        │
│                                                                       │
│ Major version upgrades (e.g., PostgreSQL 15 → 16):                │
│ ├── In-place major version upgrade available                    │
│ ├── Causes downtime (minutes to hours depending on DB size)    │
│ ├── Test on a clone first!                                      │
│ │   gcloud sql instances clone my-instance test-upgrade        │
│ │   gcloud sql instances patch test-upgrade \                   │
│ │     --database-version=POSTGRES_16                            │
│ └── Or blue-green: Create new instance → migrate → switch     │
│                                                                       │
│ Self-service maintenance:                                          │
│ ├── Console → Instance → Maintenance → Reschedule              │
│ ├── Can reschedule pending maintenance                         │
│ └── Can trigger maintenance immediately (if pending)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Import & Export

```
┌─────────────────────────────────────────────────────────────────────┐
│           IMPORT & EXPORT                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ EXPORT:                                                              │
│ ├── Format: SQL dump or CSV                                      │
│ ├── Destination: Cloud Storage bucket (gs://bucket/file.sql)    │
│ ├── Scope: Entire instance, specific database, or specific table│
│ ├── Console → SQL → Instance → Export                           │
│ ├── ⚡ Export can impact performance — run during low-traffic     │
│ └── Use serverless export (flag) to avoid impact on instance   │
│                                                                       │
│ CLI:                                                                │
│ gcloud sql export sql my-instance gs://bucket/backup.sql \       │
│   --database=mydb                                                 │
│                                                                       │
│ gcloud sql export csv my-instance gs://bucket/data.csv \         │
│   --database=mydb \                                               │
│   --query="SELECT * FROM users WHERE active = true"              │
│                                                                       │
│ IMPORT:                                                              │
│ ├── Format: SQL dump or CSV                                      │
│ ├── Source: Cloud Storage bucket                                │
│ ├── ⚠️ Must grant Cloud SQL service account access to bucket    │
│ │   Service account: PROJECT_NUMBER@gcp-sa-cloud-sql.iam.      │
│ │   gserviceaccount.com                                         │
│ │   Role: storage.objectViewer on the bucket                   │
│ ├── Console → SQL → Instance → Import                           │
│ └── For large databases: Consider DMS (Database Migration Svc) │
│                                                                       │
│ CLI:                                                                │
│ gcloud sql import sql my-instance gs://bucket/backup.sql \       │
│   --database=mydb                                                 │
│                                                                       │
│ Migration from external sources:                                   │
│ ├── Small DB (< 10 GB): Export SQL dump → upload to GCS → import│
│ ├── Large DB: Use Database Migration Service (DMS)              │
│ │   ├── Continuous replication from source to Cloud SQL         │
│ │   ├── Minimal downtime migration                              │
│ │   ├── Supports: MySQL, PostgreSQL, SQL Server                │
│ │   └── Source: On-prem, AWS RDS, Azure, other Cloud SQL       │
│ └── Very large DB: Use DMS + VPN/Interconnect for speed        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Database Flags & Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│           DATABASE FLAGS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Database engine configuration parameters exposed by          │
│       Cloud SQL. Equivalent to editing my.cnf (MySQL) or          │
│       postgresql.conf (PostgreSQL).                                │
│                                                                       │
│ Console → SQL → Instance → Edit → Flags                          │
│   [Add a database flag]                                            │
│   Flag: [slow_query_log]    Value: [on]                           │
│                                                                       │
│ Important MySQL flags:                                              │
│ ├── slow_query_log: on (enable slow query logging)              │
│ ├── long_query_time: 1 (queries > 1 sec are logged)            │
│ ├── max_connections: 1000 (max concurrent connections)          │
│ ├── innodb_buffer_pool_size: auto (usually don't change)       │
│ ├── log_bin_trust_function_creators: on (for replication)      │
│ └── general_log: on (⚠️ logs ALL queries — heavy overhead)      │
│                                                                       │
│ Important PostgreSQL flags:                                        │
│ ├── log_min_duration_statement: 1000 (log queries > 1 sec)     │
│ ├── max_connections: 500 (max concurrent connections)           │
│ │   ⚡ Each connection uses ~5-10 MB RAM!                        │
│ │   ⚡ Use connection pooling (PgBouncer) instead of raising    │
│ ├── shared_buffers: auto (25% of RAM, usually fine)            │
│ ├── work_mem: 4MB (per-operation sort memory)                  │
│ ├── pgaudit.log: all (audit logging via pgAudit extension)    │
│ ├── cloudsql.iam_authentication: on (enable IAM auth)         │
│ └── pg_stat_statements.track: all (query performance stats)   │
│                                                                       │
│ ⚡ Some flags require a restart to take effect!                    │
│   Console shows "Requires restart" next to those flags.          │
│                                                                       │
│ CLI:                                                                │
│ gcloud sql instances patch my-instance \                          │
│   --database-flags=slow_query_log=on,long_query_time=1           │
│                                                                       │
│ ⚠️ Setting --database-flags REPLACES all existing flags!          │
│   Always include ALL flags you want, not just the new one.      │
│   To clear all flags: --database-flags=""                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Monitoring & Performance

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING & QUERY INSIGHTS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Built-in monitoring (Console → SQL → Instance → Overview):        │
│ ├── CPU utilization (%)                                           │
│ ├── Memory utilization (%)                                       │
│ ├── Storage usage (GB)                                           │
│ ├── Connections count                                            │
│ ├── Read/Write operations                                        │
│ ├── Network bytes in/out                                        │
│ └── Replication lag (for replicas)                               │
│                                                                       │
│ Query Insights (Console → SQL → Instance → Query Insights):      │
│ ├── Top queries by load (CPU, I/O, lock wait)                  │
│ ├── Query execution count and latency                          │
│ ├── Query plan visualization                                    │
│ ├── Tag queries by application (add comments to SQL)           │
│ ├── Filter by database, user, client IP                        │
│ └── ⚡ Enable at instance creation or later (no restart needed)  │
│                                                                       │
│ ┌───────────────────────────────────────────────────────────┐     │
│ │ Query Insights Dashboard                                   │     │
│ │                                                            │     │
│ │ Top Queries (by total execution time):                     │     │
│ │ ┌──────────────────────────────┬───────┬──────┬──────┐    │     │
│ │ │ Query                        │ Count │ Avg  │ Total│    │     │
│ │ │                              │       │ (ms) │ (sec)│    │     │
│ │ ├──────────────────────────────┼───────┼──────┼──────┤    │     │
│ │ │ SELECT * FROM orders WHERE..│ 50K   │ 45   │ 2250 │    │     │
│ │ │ INSERT INTO events (...     │ 120K  │ 5    │ 600  │    │     │
│ │ │ UPDATE users SET last_login │ 30K   │ 12   │ 360  │    │     │
│ │ └──────────────────────────────┴───────┴──────┴──────┘    │     │
│ └───────────────────────────────────────────────────────────┘     │
│                                                                       │
│ Performance tips:                                                   │
│ ├── CPU > 80% sustained: Upgrade machine type                   │
│ ├── Memory > 90%: Upgrade machine type or reduce connections    │
│ ├── Storage > 85%: Let auto-increase work or increase manually │
│ ├── Connections near max: Use connection pooling                │
│ │   (PgBouncer for PG, ProxySQL for MySQL)                     │
│ ├── High replication lag: Upgrade replica machine type          │
│ └── Slow queries: Add indexes, optimize queries (EXPLAIN)      │
│                                                                       │
│ Cloud Monitoring alerts:                                           │
│ ├── CPU utilization > 80% for 5 minutes                        │
│ ├── Memory utilization > 90%                                    │
│ ├── Storage usage > 85%                                         │
│ ├── Replication lag > 10 seconds                                │
│ └── Failed connections / second > threshold                    │
│                                                                       │
│ AWS equivalent: RDS Performance Insights, Enhanced Monitoring    │
│ Azure equivalent: Query Performance Insight, Azure Monitor       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Terraform & CLI

### Terraform

```hcl
# === PostgreSQL Instance with HA ===
resource "google_sql_database_instance" "primary" {
  name                = "webapp-db-prod"
  database_version    = "POSTGRES_16"
  region              = "asia-south1"
  deletion_protection = true   # Prevent accidental deletion

  settings {
    tier              = "db-custom-4-15360"   # 4 vCPUs, 15 GB RAM
    edition           = "ENTERPRISE"
    availability_type = "REGIONAL"   # HA (REGIONAL) or single zone (ZONAL)

    # Storage
    disk_type       = "PD_SSD"
    disk_size       = 100   # GB
    disk_autoresize = true
    disk_autoresize_limit = 500   # Max auto-resize to 500 GB

    # Backup
    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      start_time                     = "02:00"   # UTC
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
        retention_unit   = "COUNT"
      }
      location = "asia"   # Multi-region backup location
    }

    # Maintenance
    maintenance_window {
      day          = 7   # Sunday (1=Mon, 7=Sun)
      hour         = 2   # 02:00 UTC
      update_track = "stable"   # or "canary" for early updates
    }

    # Deny maintenance period
    deny_maintenance_period {
      start_date = "2026-11-25"
      end_date   = "2026-12-02"
      time       = "00:00:00"
    }

    # IP configuration
    ip_configuration {
      ipv4_enabled    = false   # No public IP
      private_network = google_compute_network.vpc.id
      require_ssl     = true

      # If you need public IP for dev:
      # ipv4_enabled = true
      # authorized_networks {
      #   name  = "office"
      #   value = "203.0.113.0/24"
      # }
    }

    # Database flags
    database_flags {
      name  = "log_min_duration_statement"
      value = "1000"   # Log queries > 1 second
    }
    database_flags {
      name  = "max_connections"
      value = "500"
    }
    database_flags {
      name  = "cloudsql.iam_authentication"
      value = "on"
    }

    # Query Insights
    insights_config {
      query_insights_enabled  = true
      query_plans_per_minute  = 5
      query_string_length     = 4096
      record_application_tags = true
      record_client_address   = true
    }

    user_labels = {
      environment = "prod"
      team        = "backend"
      managed-by  = "terraform"
    }
  }

  depends_on = [google_service_networking_connection.private_vpc]
}

# Private Services Access (required for Private IP)
resource "google_compute_global_address" "private_ip_range" {
  name          = "cloud-sql-ip-range"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.vpc.id
}

resource "google_service_networking_connection" "private_vpc" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_range.name]
}

# === Database ===
resource "google_sql_database" "app_db" {
  name     = "webapp"
  instance = google_sql_database_instance.primary.name
}

# === Database User ===
resource "google_sql_user" "app_user" {
  name     = "webapp-user"
  instance = google_sql_database_instance.primary.name
  password = var.db_password   # Use Secret Manager in production!
}

# IAM Database User (passwordless)
resource "google_sql_user" "iam_user" {
  name     = "app-sa@my-project.iam.gserviceaccount.com"
  instance = google_sql_database_instance.primary.name
  type     = "CLOUD_IAM_SERVICE_ACCOUNT"
}

# === Read Replica (same region) ===
resource "google_sql_database_instance" "read_replica" {
  name                 = "webapp-db-replica-1"
  master_instance_name = google_sql_database_instance.primary.name
  database_version     = "POSTGRES_16"
  region               = "asia-south1"

  replica_configuration {
    failover_target = false
  }

  settings {
    tier              = "db-custom-4-15360"
    availability_type = "ZONAL"

    disk_type       = "PD_SSD"
    disk_size       = 100
    disk_autoresize = true

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }

    insights_config {
      query_insights_enabled = true
    }
  }
}

# === Cross-Region Read Replica (for DR) ===
resource "google_sql_database_instance" "dr_replica" {
  name                 = "webapp-db-dr-us"
  master_instance_name = google_sql_database_instance.primary.name
  database_version     = "POSTGRES_16"
  region               = "us-central1"   # Different region for DR

  replica_configuration {
    failover_target = false
  }

  settings {
    tier              = "db-custom-2-7680"   # Can be smaller
    availability_type = "ZONAL"

    disk_type       = "PD_SSD"
    disk_size       = 100
    disk_autoresize = true

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.us_vpc.id
    }
  }
}

# Output connection info
output "db_private_ip" {
  value = google_sql_database_instance.primary.private_ip_address
}

output "db_connection_name" {
  value = google_sql_database_instance.primary.connection_name
  # Format: project:region:instance-name
  # Used by Auth Proxy and Connectors
}
```

### gcloud CLI Reference

```bash
# =============================================
# INSTANCE MANAGEMENT
# =============================================

# Create PostgreSQL instance with HA
gcloud sql instances create webapp-db-prod \
  --database-version=POSTGRES_16 \
  --tier=db-custom-4-15360 \
  --region=asia-south1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=100GB \
  --storage-auto-increase \
  --storage-auto-increase-limit=500GB \
  --backup-start-time=02:00 \
  --enable-point-in-time-recovery \
  --retained-backups-count=30 \
  --enable-bin-log \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=2 \
  --network=projects/my-project/global/networks/default \
  --no-assign-ip \
  --require-ssl \
  --database-flags=log_min_duration_statement=1000,max_connections=500

# Create MySQL instance
gcloud sql instances create mysql-db \
  --database-version=MYSQL_8_0 \
  --tier=db-custom-2-7680 \
  --region=asia-south1 \
  --root-password=SECURE_PASSWORD

# List instances
gcloud sql instances list

# Describe instance
gcloud sql instances describe webapp-db-prod

# Resize (change machine type — causes restart)
gcloud sql instances patch webapp-db-prod \
  --tier=db-custom-8-30720

# Change storage size
gcloud sql instances patch webapp-db-prod \
  --storage-size=200GB

# =============================================
# DATABASES & USERS
# =============================================

# Create database
gcloud sql databases create webapp --instance=webapp-db-prod

# List databases
gcloud sql databases list --instance=webapp-db-prod

# Create user
gcloud sql users create webapp-user \
  --instance=webapp-db-prod \
  --password=SECURE_PASSWORD

# Create IAM user (passwordless)
gcloud sql users create sa@project.iam.gserviceaccount.com \
  --instance=webapp-db-prod \
  --type=CLOUD_IAM_SERVICE_ACCOUNT

# List users
gcloud sql users list --instance=webapp-db-prod

# =============================================
# REPLICAS
# =============================================

# Create read replica (same region)
gcloud sql instances create webapp-replica-1 \
  --master-instance-name=webapp-db-prod \
  --tier=db-custom-4-15360 \
  --region=asia-south1

# Create cross-region replica (DR)
gcloud sql instances create webapp-dr-us \
  --master-instance-name=webapp-db-prod \
  --tier=db-custom-2-7680 \
  --region=us-central1

# Promote replica to standalone (⚠️ irreversible!)
gcloud sql instances promote-replica webapp-dr-us

# =============================================
# BACKUPS
# =============================================

# Create on-demand backup
gcloud sql backups create --instance=webapp-db-prod

# List backups
gcloud sql backups list --instance=webapp-db-prod

# Restore from backup
gcloud sql backups restore BACKUP_ID \
  --restore-instance=webapp-db-prod

# Point-in-time recovery (clone to new instance)
gcloud sql instances clone webapp-db-prod webapp-db-restored \
  --point-in-time="2026-05-17T10:30:00.000Z"

# =============================================
# EXPORT & IMPORT
# =============================================

# Export to Cloud Storage
gcloud sql export sql webapp-db-prod gs://my-bucket/backup.sql \
  --database=webapp

gcloud sql export csv webapp-db-prod gs://my-bucket/users.csv \
  --database=webapp \
  --query="SELECT id, email FROM users"

# Import from Cloud Storage
gcloud sql import sql webapp-db-prod gs://my-bucket/backup.sql \
  --database=webapp

# =============================================
# CONNECT
# =============================================

# Connect via Auth Proxy
cloud-sql-proxy my-project:asia-south1:webapp-db-prod --port=5432

# Direct connect via gcloud (quick testing)
gcloud sql connect webapp-db-prod --user=postgres --database=webapp

# =============================================
# MAINTENANCE
# =============================================

# Trigger manual failover (test HA)
gcloud sql instances failover webapp-db-prod

# Reschedule maintenance
gcloud sql instances reschedule-maintenance webapp-db-prod \
  --reschedule-type=NEXT_AVAILABLE_WINDOW

# Major version upgrade
gcloud sql instances patch webapp-db-prod \
  --database-version=POSTGRES_17

# =============================================
# DATABASE FLAGS
# =============================================

# Set flags (⚠️ replaces ALL existing flags!)
gcloud sql instances patch webapp-db-prod \
  --database-flags=log_min_duration_statement=1000,\
max_connections=500,cloudsql.iam_authentication=on

# Clear all flags
gcloud sql instances patch webapp-db-prod --database-flags=""

# =============================================
# DELETE
# =============================================

# Delete instance (must delete replicas first)
gcloud sql instances delete webapp-db-prod
# ⚡ Instance name cannot be reused for ~1 week
```

---

## Part 15: Real-World Patterns

### Startup

```
┌─────────────────────────────────────────────────────────────────────┐
│ STARTUP (5-10 developers)                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance:                                                            │
│ ├── PostgreSQL 16 (most features, best for new projects)        │
│ ├── db-custom-2-7680 (2 vCPUs, 7.5 GB RAM)                     │
│ ├── 50 GB SSD with auto-increase                                │
│ ├── Single zone (save cost, acceptable risk)                    │
│ └── Public IP + Auth Proxy for dev, Private IP if in VPC       │
│                                                                       │
│ Backups:                                                             │
│ ├── Automated daily (7-day retention)                           │
│ ├── PITR enabled (7-day log retention)                          │
│ └── On-demand backup before major migrations                   │
│                                                                       │
│ Replicas: None (add when reads become bottleneck)                │
│ Encryption: Google-managed (free)                                 │
│ Connection: Auth Proxy + Cloud Run connector                     │
│                                                                       │
│ Monthly cost: ~$100-150                                            │
│                                                                       │
│ ⚡ Start with db-f1-micro for dev/test (free tier!).               │
│   Upgrade to custom machine for production.                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Mid-Size

```
┌─────────────────────────────────────────────────────────────────────┐
│ MID-SIZE (50-100 developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance:                                                            │
│ ├── PostgreSQL 16, Enterprise edition                            │
│ ├── db-custom-8-30720 (8 vCPUs, 30 GB RAM)                     │
│ ├── 200 GB SSD with auto-increase (limit: 1 TB)                │
│ ├── HA enabled (REGIONAL) ✅                                     │
│ └── Private IP only (no public IP)                              │
│                                                                       │
│ Replicas:                                                            │
│ ├── 1-2 same-region read replicas (scale reads)                │
│ ├── 1 cross-region replica (DR in us-central1)                 │
│ └── App uses connection pooling (PgBouncer)                    │
│                                                                       │
│ Backups:                                                             │
│ ├── Automated daily (30-day retention)                          │
│ ├── PITR enabled (7-day log retention)                          │
│ ├── Cross-region backup location                                │
│ └── Tested restore procedures quarterly                        │
│                                                                       │
│ Security:                                                            │
│ ├── IAM database authentication for service accounts           │
│ ├── SSL-only connections enforced                               │
│ ├── Database flags tuned for workload                          │
│ ├── Query Insights enabled                                     │
│ └── Separate users per application/service                     │
│                                                                       │
│ Connection: Auth Proxy sidecar in GKE, Connector in Cloud Run  │
│                                                                       │
│ Monthly cost: ~$800-1,500                                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Enterprise

```
┌─────────────────────────────────────────────────────────────────────┐
│ ENTERPRISE (500+ developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance:                                                            │
│ ├── PostgreSQL 16, Enterprise Plus edition                      │
│ │   (data cache, faster failover, advanced HA)                  │
│ ├── db-custom-32-122880 (32 vCPUs, 120 GB RAM)                 │
│ ├── 1 TB SSD with auto-increase                                │
│ ├── HA enabled (REGIONAL) ✅                                     │
│ ├── Private IP only + VPC Service Controls                     │
│ └── CMEK encryption (Cloud KMS)                                 │
│                                                                       │
│ Architecture:                                                        │
│ ├── Separate instances per service (microservices pattern)      │
│ ├── 3-5 read replicas per primary (scale reads)                │
│ ├── Cross-region replicas for DR + geo-local reads             │
│ ├── Cascading replicas for multi-region                        │
│ ├── Connection pooling (PgBouncer sidecar or external)         │
│ └── Database per service, shared nothing                       │
│                                                                       │
│ Backups & DR:                                                        │
│ ├── Multi-region backup storage                                 │
│ ├── 90-day backup retention                                    │
│ ├── PITR with 7-day log retention                              │
│ ├── Automated DR drills quarterly                               │
│ ├── Cross-region replica with automated promotion runbook      │
│ └── RPO < 1 min, RTO < 15 min                                 │
│                                                                       │
│ Security:                                                            │
│ ├── CMEK on all instances (KMS key per data classification)    │
│ ├── IAM-only authentication (no password-based logins)         │
│ ├── VPC Service Controls perimeter                              │
│ ├── pgAudit enabled (all DDL/DML logged)                       │
│ ├── Cloud Audit Logs for admin operations                      │
│ └── Deny maintenance periods for critical business dates       │
│                                                                       │
│ Monitoring:                                                         │
│ ├── CPU/Memory/Storage/Connection alerts                       │
│ ├── Replication lag alerts (> 5 sec = warning)                 │
│ ├── Query Insights + slow query dashboards                     │
│ ├── Custom Grafana dashboards                                   │
│ └── PagerDuty integration for on-call                          │
│                                                                       │
│ Consider AlloyDB: If Cloud SQL PostgreSQL hits performance      │
│ limits, AlloyDB offers 4x throughput + columnar engine.        │
│ Consider Spanner: If you need global scale + 99.999% SLA.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌──────────────────────────────────────────────────────────────────────┐
│ CLOUD SQL — QUICK REFERENCE                                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ Engines: MySQL (5.7, 8.0, 8.4) | PostgreSQL (13-17) | SQL Server   │
│ Max: 96 vCPUs, 624 GB RAM, 64 TB storage                           │
│ Editions: Enterprise | Enterprise Plus (data cache, faster HA)      │
│                                                                        │
│ HA: REGIONAL (sync replication, auto-failover, ~60s / ~10s E+)     │
│ Read Replicas: Up to 10, async, same or cross-region, promotable   │
│ Backups: Automated daily + on-demand + PITR (up to 365 days)       │
│                                                                        │
│ Connectivity:                                                        │
│ ├── Public IP + authorized networks (dev only)                      │
│ ├── Private IP via PSA (production ✅)                               │
│ ├── Cloud SQL Auth Proxy (encrypted tunnel ✅)                      │
│ └── Connectors (in-process, serverless ✅)                           │
│                                                                        │
│ Security: Encryption (rest + transit) | CMEK | IAM auth | SSL      │
│ Monitoring: Query Insights | Cloud Monitoring metrics + alerts      │
│                                                                        │
│ Key limitations:                                                     │
│ ├── Cannot change engine after creation                              │
│ ├── Cannot shrink storage — ever                                    │
│ ├── Instance name reuse blocked for ~1 week after deletion          │
│ ├── Cannot change region after creation                              │
│ ├── HA standby does NOT serve read traffic                          │
│ └── Promoted replicas cannot be demoted back                        │
│                                                                        │
│ AWS ↔ GCP mapping:                                                     │
│ ├── RDS → Cloud SQL                                                   │
│ ├── RDS Multi-AZ → Cloud SQL HA (REGIONAL)                           │
│ ├── RDS Read Replica → Cloud SQL Read Replica                        │
│ ├── RDS Proxy → Auth Proxy (different — proxy = auth tunnel)        │
│ ├── RDS Performance Insights → Query Insights                        │
│ ├── RDS Automated Backups → Automated Backups                        │
│ ├── RDS PITR → PITR (same concept)                                   │
│ └── DMS → Database Migration Service                                  │
│                                                                        │
│ Azure ↔ GCP mapping:                                                   │
│ ├── Azure SQL Database → Cloud SQL (SQL Server)                      │
│ ├── Azure DB for MySQL → Cloud SQL (MySQL)                           │
│ ├── Azure DB for PostgreSQL → Cloud SQL (PostgreSQL)                 │
│ ├── Zone-redundant → Cloud SQL HA                                     │
│ ├── Read replicas → Read replicas                                     │
│ ├── Private Endpoint → Private IP (PSA)                               │
│ └── Query Performance Insight → Query Insights                        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Instances

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING INSTANCES FROM CONSOLE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ === CHANGE MACHINE TYPE ===                                         │
│                                                                       │
│ Console → SQL → select instance → Edit                            │
│                                                                       │
│ 1. Under "Machine configuration", choose new machine type         │
│    e.g., db-custom-4-15360 → db-custom-8-30720                   │
│ 2. Click "Save"                                                   │
│ 3. ⚠️ Instance will RESTART (~1-2 minutes downtime)               │
│                                                                       │
│ ⚡ With HA enabled, the restart is handled via failover,           │
│   reducing downtime to seconds.                                   │
│ ⚡ Schedule machine type changes during maintenance windows        │
│   or low-traffic periods.                                         │
│                                                                       │
│ === ENABLE / DISABLE HA ===                                         │
│                                                                       │
│ Console → SQL → select instance → Edit                            │
│                                                                       │
│ 1. Under "Machine configuration" → "Zonal availability"          │
│    ○ Multiple zones (HA) — creates a standby in another zone    │
│    ○ Single zone — removes the standby                           │
│ 2. Click "Save"                                                   │
│                                                                       │
│ ⚡ Enabling HA: Provisions a standby instance (takes a few min).  │
│   Doubles compute + storage cost.                                 │
│ ⚡ Disabling HA: Removes the standby (⚠️ no more auto-failover). │
│   Only do this for dev/test or if switching to a different DR     │
│   strategy.                                                       │
│                                                                       │
│ === MANAGE DATABASE FLAGS ===                                       │
│                                                                       │
│ Console → SQL → select instance → Edit → Flags                   │
│                                                                       │
│ 1. Click "Add a database flag"                                    │
│ 2. Search for the flag name (e.g., "max_connections")            │
│ 3. Enter the value (e.g., "500")                                 │
│ 4. Repeat for additional flags                                    │
│ 5. Click "Save"                                                   │
│                                                                       │
│ ⚡ Some flags require a restart — Console shows a restart icon    │
│   next to those flags.                                            │
│ ⚡ To remove a flag: Click the "X" next to it and Save.           │
│ ⚡ Removing a flag resets it to its default value.                 │
│                                                                       │
│ === DELETE A READ REPLICA ===                                       │
│                                                                       │
│ Console → SQL → select the REPLICA instance                       │
│   (replicas appear as separate entries in the instance list)     │
│                                                                       │
│ 1. Click "Delete" at the top (or ⋮ → Delete)                    │
│ 2. Type the instance name to confirm                              │
│ 3. Click "Delete"                                                 │
│                                                                       │
│ ⚡ Deleting a replica does NOT affect the primary.                 │
│ ⚡ If you have cascading replicas (Primary → A → B), deleting    │
│   replica A also breaks replication to B.                        │
│                                                                       │
│ === DELETE A CLOUD SQL INSTANCE ===                                 │
│                                                                       │
│ Console → SQL → select instance → Delete (or ⋮ → Delete)        │
│                                                                       │
│ 1. Delete ALL read replicas first — you cannot delete a          │
│    primary that still has replicas.                              │
│ 2. Type the instance name to confirm                              │
│ 3. Click "Delete"                                                 │
│                                                                       │
│ ⚠️ DELETION PROTECTION:                                            │
│ ├── If deletion_protection = true (Terraform) or enabled via    │
│ │   Console, you CANNOT delete the instance.                    │
│ ├── Console → SQL → Instance → Edit → scroll to bottom         │
│ │   ☐ Enable deletion protection  ← uncheck this first         │
│ ├── CLI: gcloud sql instances patch my-instance \               │
│ │        --no-deletion-protection                                │
│ ├── ⚡ ALWAYS enable deletion protection on production!           │
│ │   Prevents accidental deletion by a tired engineer at 3 AM.  │
│ └── Terraform: deletion_protection = true (set in resource)    │
│                                                                       │
│ ⚡ After deletion, the instance name is RESERVED for ~1 week.     │
│   You cannot create a new instance with the same name.           │
│ ⚡ All automated backups are deleted with the instance.            │
│   On-demand backups are ALSO deleted.                            │
│ ⚡ If you need the data: Export or take a final backup FIRST!     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Connection Pooling

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECTION POOLING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ WHY IT MATTERS:                                                     │
│                                                                       │
│ Problem: Each database connection uses RAM (~5-10 MB for PG).     │
│ Cloud Run / Cloud Functions can spin up hundreds of instances,    │
│ each opening its own connection → you hit max_connections fast.  │
│                                                                       │
│ Without pooling:                                                    │
│                                                                       │
│ ┌──────────────┐                                                   │
│ │ Cloud Run    │──conn 1──►┐                                       │
│ │ instance 1   │           │                                       │
│ ├──────────────┤           │  ┌──────────────┐                     │
│ │ Cloud Run    │──conn 2──►├──│ Cloud SQL    │ max_connections=100│
│ │ instance 2   │           │  │ (PostgreSQL) │ ⚠️ EXHAUSTED!       │
│ ├──────────────┤           │  └──────────────┘                     │
│ │ ... 98 more  │──conn 100►┘                                       │
│ │ instances    │                                                   │
│ ├──────────────┤                                                   │
│ │ Cloud Run    │──conn 101►  ❌ ERROR: too many connections       │
│ │ instance 101 │                                                   │
│ └──────────────┘                                                   │
│                                                                       │
│ With pooling:                                                       │
│                                                                       │
│ ┌──────────────┐                                                   │
│ │ Cloud Run    │──►┐                                               │
│ │ instance 1   │   │  ┌──────────────┐    ┌──────────────┐        │
│ ├──────────────┤   ├──│ Connection   │──►│ Cloud SQL    │        │
│ │ Cloud Run    │──►│  │ Pooler       │    │ (PostgreSQL) │        │
│ │ instance 2   │   │  │ (PgBouncer)  │    │              │        │
│ ├──────────────┤   │  │ 200 clients  │    │ Only 20 real │        │
│ │ ... 200 more │──►┘  │  → 20 conns  │    │ connections  │        │
│ └──────────────┘      └──────────────┘    └──────────────┘        │
│                                                                       │
│ CLOUD SQL AUTH PROXY CONNECTION LIMITS:                              │
│ ├── Auth Proxy does NOT pool connections — it's a tunnel only    │
│ ├── Each app connection through the proxy = one DB connection   │
│ ├── You still need a pooler for high-concurrency workloads      │
│ └── Proxy handles auth + encryption, not connection management  │
│                                                                       │
│ PGBOUNCER (PostgreSQL):                                             │
│ ├── Lightweight connection pooler — sits between app and DB     │
│ ├── Modes:                                                       │
│ │   ├── Session: 1 client = 1 server conn (like no pooling)    │
│ │   ├── Transaction: Server conn reused between transactions   │
│ │   │   ⚡ Best for most apps — recommended                     │
│ │   └── Statement: Server conn reused per statement (limited) │
│ ├── Deploy as:                                                   │
│ │   ├── Sidecar container in GKE pod                           │
│ │   ├── Standalone VM or Cloud Run service                     │
│ │   └── Cloud SQL does NOT include built-in PgBouncer          │
│ │       (unlike some managed PG providers)                     │
│ └── AWS equivalent: RDS Proxy (managed pooler — GCP has none)  │
│                                                                       │
│ For MySQL: ProxySQL or application-level pooling                  │
│ (HikariCP for Java, SQLAlchemy pool for Python, etc.)            │
│                                                                       │
│ BRIEF EXAMPLE — PgBouncer config (pgbouncer.ini):                 │
│                                                                       │
│   [databases]                                                       │
│   mydb = host=10.0.1.5 port=5432 dbname=webapp                   │
│                                                                       │
│   [pgbouncer]                                                       │
│   listen_addr = 0.0.0.0                                            │
│   listen_port = 6432                                               │
│   pool_mode = transaction                                          │
│   max_client_conn = 500                                            │
│   default_pool_size = 20                                           │
│                                                                       │
│ Your app connects to PgBouncer on port 6432 instead of directly  │
│ to Cloud SQL on port 5432. PgBouncer manages a small pool of     │
│ real connections and multiplexes client requests over them.       │
│                                                                       │
│ ⚡ Rule of thumb: If your app has more than ~50 concurrent        │
│   connections (or uses serverless compute), set up pooling.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Query Insights

```
┌─────────────────────────────────────────────────────────────────────┐
│           QUERY INSIGHTS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ WHAT IS IT?                                                         │
│                                                                       │
│ Query Insights is a built-in Cloud SQL feature that helps you     │
│ find and fix slow queries. Think of it as a performance profiler  │
│ for your database — it shows which queries use the most CPU,     │
│ which ones wait on I/O, and which ones are called most often.    │
│                                                                       │
│ ⚡ Available for MySQL and PostgreSQL (not SQL Server).             │
│ ⚡ No additional cost — included with Cloud SQL.                   │
│ ⚡ Minimal performance overhead (~1-2%).                            │
│                                                                       │
│ WHAT IT SHOWS:                                                      │
│ ├── Top queries ranked by total execution time or CPU load       │
│ ├── Query execution count, average latency, total time          │
│ ├── Wait events (CPU, I/O, lock wait, etc.)                    │
│ ├── Query plan visualization (EXPLAIN plan)                    │
│ ├── Queries tagged by application (if you add SQL comments)    │
│ ├── Filter by: database, user, client IP address               │
│ └── Historical data (up to 7 days)                              │
│                                                                       │
│ HOW TO ENABLE:                                                      │
│                                                                       │
│ Option A — During instance creation:                                │
│   Console → SQL → Create Instance → Query Insights              │
│   ☑ Enable Query Insights                                        │
│                                                                       │
│ Option B — On an existing instance:                                 │
│   Console → SQL → select instance → Edit                        │
│   Scroll to "Query Insights"                                     │
│   ☑ Enable Query Insights                                        │
│   ☑ Record application tags (optional but recommended)          │
│   ☑ Record client IP address (optional)                         │
│   Query string length: [4096] (default 1024, increase for       │
│     longer queries)                                              │
│   Click "Save"                                                   │
│                                                                       │
│ ⚡ No restart required! Takes effect within a few minutes.         │
│                                                                       │
│ HOW TO VIEW:                                                        │
│                                                                       │
│ Console → SQL → select instance → Query Insights                 │
│                                                                       │
│ ┌───────────────────────────────────────────────────────────┐     │
│ │ Query Insights Dashboard                                   │     │
│ │                                                            │     │
│ │ Database load by query (last 6 hours):                     │     │
│ │ ┌──────────────────────────────────────────────────────┐  │     │
│ │ │ ████████████████████  SELECT * FROM orders WHERE... │  │     │
│ │ │ ███████              INSERT INTO events (...)        │  │     │
│ │ │ ████                 UPDATE users SET last_login...  │  │     │
│ │ └──────────────────────────────────────────────────────┘  │     │
│ │                                                            │     │
│ │ Click any query → see execution plan, latency breakdown,  │     │
│ │ and optimization suggestions.                              │     │
│ │                                                            │     │
│ │ Filter by: [All databases ▼] [All users ▼] [Time range ▼]│     │
│ └───────────────────────────────────────────────────────────┘     │
│                                                                       │
│ TAGGING QUERIES (optional):                                         │
│ Add a SQL comment to tag queries by application or feature:       │
│                                                                       │
│   /* controller=OrderController,action=list */                    │
│   SELECT * FROM orders WHERE user_id = $1;                       │
│                                                                       │
│ Query Insights groups these tags so you can see which part of    │
│ your app generates the most database load.                       │
│                                                                       │
│ AWS equivalent: RDS Performance Insights                           │
│ Azure equivalent: Query Performance Insight                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Additional gcloud Commands

```bash
# =============================================
# PATCH INSTANCE (change configuration)
# =============================================

# Change machine type (⚠️ causes restart / brief downtime)
gcloud sql instances patch my-instance \
  --tier=db-custom-8-30720

# Enable HA (adds a standby in another zone)
gcloud sql instances patch my-instance \
  --availability-type=REGIONAL

# Disable HA (removes the standby — ⚠️ no more auto-failover)
gcloud sql instances patch my-instance \
  --availability-type=ZONAL

# Update maintenance window (Sunday at 3 AM UTC)
gcloud sql instances patch my-instance \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=3

# Enable deletion protection (recommended for production)
gcloud sql instances patch my-instance \
  --deletion-protection

# Disable deletion protection (required before deleting)
gcloud sql instances patch my-instance \
  --no-deletion-protection

# Enable Query Insights
gcloud sql instances patch my-instance \
  --insights-config-query-insights-enabled \
  --insights-config-record-application-tags \
  --insights-config-record-client-address

# =============================================
# DELETE INSTANCE
# =============================================

# Step 1: Delete all read replicas first
gcloud sql instances delete my-replica-1 --quiet
gcloud sql instances delete my-replica-2 --quiet

# Step 2: Disable deletion protection if enabled
gcloud sql instances patch my-instance \
  --no-deletion-protection

# Step 3: Delete the primary instance
gcloud sql instances delete my-instance

# ⚡ You'll be prompted to confirm — type "y" or use --quiet
# ⚡ Instance name is RESERVED for ~1 week after deletion
# ⚡ All backups (automated + on-demand) are deleted

# =============================================
# PROMOTE READ REPLICA
# =============================================

# Promote a read replica to a standalone read-write instance
# ⚠️ This is IRREVERSIBLE — the replica becomes independent!
gcloud sql instances promote-replica my-dr-replica

# Common use cases:
# ├── DR failover: Primary region is down, promote cross-region
# │   replica as the new primary
# ├── Region migration: Move your primary to a new region
# └── Split workload: Make the replica an independent instance
#
# After promotion:
# ├── The instance is now a standalone primary (read + write)
# ├── Replication from the old primary is permanently broken
# ├── You need to update your application connection strings
# ├── Set up HA on the promoted instance if needed:
#     gcloud sql instances patch my-dr-replica \
#       --availability-type=REGIONAL
# └── Create new replicas under the promoted instance if needed
```

---

## What's Next?

Continue to **Chapter 25: Cloud Spanner** → `25-cloud-spanner.md`
