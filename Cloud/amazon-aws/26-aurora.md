# Chapter 26: Amazon Aurora

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Aurora Fundamentals](#part-1-aurora-fundamentals)
- [Part 2: Creating an Aurora Cluster (Full Portal Walkthrough)](#part-2-creating-an-aurora-cluster-full-portal-walkthrough)
- [Part 3: Aurora Serverless v2](#part-3-aurora-serverless-v2)
- [Part 4: Aurora Global Database](#part-4-aurora-global-database)
- [Part 5: Aurora Cloning & Backtrack](#part-5-aurora-cloning--backtrack)
- [Part 6: Aurora Machine Learning & Integrations](#part-6-aurora-machine-learning--integrations)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Aurora? When Should I Pick Aurora Over Standard RDS?

If RDS is like hiring a database administrator, **Aurora is like hiring a supercharged DBA with a custom-built engine.** Aurora is AWS's own database engine that's compatible with MySQL and PostgreSQL — meaning your existing MySQL/PostgreSQL code works without changes — but it's significantly faster and more resilient.

**When to choose Aurora over standard RDS:**
- Your app needs **higher performance** (5x MySQL, 3x PostgreSQL throughput)
- You need **faster failover** (30 seconds vs 1-2 minutes for RDS Multi-AZ)
- You want **auto-scaling storage** (grows automatically up to 128 TB, no need to pre-provision)
- You want **up to 15 read replicas** (RDS allows only 5)
- You need **global databases** for worldwide users with <1 second replication

**When standard RDS is fine:**
- Small to medium workloads where Aurora's extra features aren't needed
- You need Oracle or SQL Server (Aurora only supports MySQL and PostgreSQL)
- Budget is tight (Aurora is ~20% more expensive than standard RDS)

Amazon Aurora is AWS's purpose-built relational database engine, compatible with MySQL and PostgreSQL. It provides up to 5x the throughput of MySQL and 3x of PostgreSQL, with the security, availability, and reliability of commercial databases at 1/10th the cost.

```
What you'll learn:
├── Aurora Fundamentals
│   ├── Aurora vs standard RDS
│   ├── Cluster architecture (writer + readers)
│   ├── Storage architecture (shared distributed storage)
│   └── Pricing model
├── Creating an Aurora Cluster (Full Portal Walkthrough)
├── Aurora Serverless v2
│   ├── Auto-scaling capacity (ACU)
│   └── Mixed provisioned + serverless
├── Global Database (cross-region replication)
├── Cloning (instant copy for dev/test)
├── Backtrack (rewind database in time)
├── ML integrations (SageMaker, Comprehend)
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: Aurora Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AURORA ARCHITECTURE                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Aurora cluster:                                                      │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │                                                                │  │
│ │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │  │
│ │  │ Writer       │    │ Reader 1     │    │ Reader 2     │    │  │
│ │  │ Instance     │    │ Instance     │    │ Instance     │    │  │
│ │  │ (read/write) │    │ (read-only)  │    │ (read-only)  │    │  │
│ │  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘    │  │
│ │         └────────────────────┼────────────────────┘           │  │
│ │                              │                                 │  │
│ │         ┌────────────────────┴────────────────────┐           │  │
│ │         │      Shared Distributed Storage          │           │  │
│ │         │      (6 copies across 3 AZs)             │           │  │
│ │         │      Auto-grows: 10 GB → 128 TB          │           │  │
│ │         └──────────────────────────────────────────┘           │  │
│ │                                                                │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Aurora vs Standard RDS:                                             │
│ ┌───────────────────┬──────────────────┬──────────────────────────┐│
│ │                   │ Aurora            │ Standard RDS             ││
│ ├───────────────────┼──────────────────┼──────────────────────────┤│
│ │ Storage           │ Shared, auto-grow│ Per-instance EBS volume  ││
│ │ Copies            │ 6 across 3 AZs   │ 1 (Multi-AZ: 2)         ││
│ │ Replication       │ Storage-level     │ Block-level (EBS)        ││
│ │ Failover          │ <30 seconds       │ 60-120 seconds           ││
│ │ Read Replicas     │ Up to 15          │ Up to 5                  ││
│ │ Performance       │ 5x MySQL          │ Standard                 ││
│ │ Auto-grow storage │ Yes (to 128 TB)   │ Auto-scale EBS           ││
│ │ Backtrack         │ Yes (MySQL)       │ No                       ││
│ │ Cloning           │ Instant (CoW)     │ Snapshot + restore       ││
│ │ Serverless        │ v2                │ No                       ││
│ │ Global Database   │ Yes (<1s repl)    │ Cross-region replica     ││
│ └───────────────────┴──────────────────┴──────────────────────────┘│
│                                                                       │
│ Endpoints:                                                           │
│ ├── Cluster (Writer): mydb.cluster-abc123.us-east-1.rds.amazonaws│
│ │   → Always points to the current writer instance               │
│ ├── Reader: mydb.cluster-ro-abc123.us-east-1.rds.amazonaws.com  │
│ │   → Load-balanced across all reader instances                  │
│ ├── Instance: mydb-instance-1.abc123.us-east-1.rds.amazonaws.com│
│ │   → Specific instance (avoid using directly)                   │
│ └── Custom: User-defined subset of instances                     │
│                                                                       │
│ Pricing (us-east-1):                                                │
│ ├── db.r6g.large: ~$0.29/hr ($209/month) — per instance        │
│ ├── Storage: $0.10/GB/month (auto-grows, pay for what you use) │
│ ├── I/O: $0.20 per million requests (Aurora Standard)           │
│ ├── Aurora I/O-Optimized: No I/O charges, 30% higher storage   │
│ ├── Backups: Free up to cluster storage size                     │
│ └── No charge for replication between writer and readers        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating an Aurora Cluster (Full Portal Walkthrough)

```
Console → RDS → Databases → Create database

┌─────────────────────────────────────────────────────────────────┐
│           CREATE AURORA DATABASE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Engine: ● Aurora (MySQL Compatible)                             │
│         ○ Aurora (PostgreSQL Compatible)                        │
│                                                                   │
│ Edition:                                                        │
│   ● Aurora MySQL                                                │
│   ○ Aurora MySQL Serverless (Legacy — use Serverless v2)       │
│                                                                   │
│ Engine version: [Aurora MySQL 3.06.0 (MySQL 8.0.34) ▼]        │
│ → Aurora 3.x = MySQL 8.0 compatible                           │
│ → Aurora 2.x = MySQL 5.7 compatible                           │
│                                                                   │
│ ── Templates ──                                                 │
│ ● Production  ○ Dev/Test                                       │
│                                                                   │
│ ── Settings ──                                                  │
│ DB cluster identifier: [prod-aurora-mysql]                    │
│ → Cluster-level identifier (applies to all instances)        │
│                                                                   │
│ Master username: [admin]                                       │
│ Credentials: ● AWS Secrets Manager ○ Self managed              │
│ → ⚡ Use Secrets Manager for production                       │
│                                                                   │
│ ── Instance configuration ──                                   │
│                                                                   │
│ ● Provisioned                                                   │
│ ○ Serverless v2                                                 │
│ ○ Both (mixed — some provisioned, some serverless)             │
│                                                                   │
│ Instance class: [db.r6g.large ▼]                               │
│ → Memory-optimized (r) classes recommended for Aurora         │
│                                                                   │
│ ── Availability & durability ──                                │
│ ☑ Create an Aurora Replica in a different AZ                  │
│ → Creates writer + 1 reader in different AZs                  │
│ → ⚡ Essential for production (automatic failover)            │
│                                                                   │
│ ── Connectivity ──                                              │
│ VPC: [vpc-main ▼]                                              │
│ DB subnet group: [aurora-private ▼]                           │
│ Public access: ● No                                            │
│ Security group: [sg-aurora ▼]                                 │
│                                                                   │
│ ── Cluster storage configuration ──                            │
│ ● Aurora Standard                                               │
│ ○ Aurora I/O-Optimized                                         │
│                                                                   │
│ → Standard: Pay per I/O ($0.20/million). Best for moderate    │
│   I/O workloads. Most cost-effective for <25% I/O spend.     │
│ → I/O-Optimized: No I/O charges. 30% higher storage cost.   │
│   Best when I/O costs exceed 25% of total Aurora spend.       │
│   ⚡ Can switch between modes (once per 30 days).             │
│                                                                   │
│ ── Monitoring ──                                                │
│ ☑ Performance Insights (7 days free)                          │
│ ☑ Enhanced Monitoring                                          │
│ ☑ DevOps Guru (ML-powered anomaly detection)                  │
│                                                                   │
│ ── Backup ──                                                    │
│ Backup retention: [7] days (1-35)                             │
│ ☑ Copy tags to snapshots                                      │
│                                                                   │
│ ── Backtrack (MySQL only) ──                                   │
│ ☑ Enable Backtrack                                             │
│ Target backtrack window: [24] hours                           │
│ → Rewind database to any point within this window            │
│ → ⚡ Faster than point-in-time restore (in-place)            │
│                                                                   │
│ ── Encryption ──                                                │
│ ☑ Enable encryption                                           │
│ KMS key: [aws/rds ▼]                                          │
│                                                                   │
│ ── Deletion protection ──                                      │
│ ☑ Enable deletion protection                                  │
│                                                                   │
│                    [Create database]                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Aurora Serverless v2

```
┌─────────────────────────────────────────────────────────────────────┐
│           AURORA SERVERLESS v2                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Aurora instances that automatically scale compute capacity   │
│ up and down based on demand. Pay per ACU (Aurora Capacity Unit).  │
│                                                                       │
│ ACU (Aurora Capacity Unit):                                         │
│ ├── 1 ACU ≈ 2 GB RAM + proportional CPU + networking             │
│ ├── Minimum: 0.5 ACU ($0.12/ACU-hour)                            │
│ ├── Maximum: 256 ACU                                               │
│ ├── Scales in increments of 0.5 ACU                               │
│ ├── Scales up in seconds, scales down in minutes                  │
│ └── ⚠️ Cannot scale to 0 (unlike v1 — minimum 0.5 ACU)          │
│                                                                       │
│ Configuration:                                                       │
│ Instance configuration: ○ Provisioned ● Serverless v2            │
│ Minimum ACUs: [0.5]                                                │
│ Maximum ACUs: [32]                                                 │
│                                                                       │
│ → Set minimum to handle baseline load                             │
│ → Set maximum for peak capacity                                   │
│ → ⚡ Can mix provisioned writer + serverless readers!             │
│                                                                       │
│ Mixed configuration example:                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Writer: db.r6g.xlarge (provisioned — consistent performance) │   │
│ │ Reader 1: Serverless v2 (0.5 - 32 ACU) — scales for reads   │   │
│ │ Reader 2: Serverless v2 (0.5 - 16 ACU) — dev/analytics      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ When to use Serverless v2:                                          │
│ ├── Variable/unpredictable workloads                              │
│ ├── Dev/test environments (scales down when idle)                │
│ ├── Multi-tenant applications                                     │
│ ├── Intermittent/bursty traffic                                   │
│ └── New applications where traffic patterns are unknown          │
│                                                                       │
│ When NOT to use:                                                     │
│ ├── Steady, predictable high load (provisioned is cheaper)      │
│ └── Need to scale to zero (use Lambda + DynamoDB instead)        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Aurora Global Database

```
Console → RDS → Databases → Select cluster → Actions → Add AWS Region

┌─────────────────────────────────────────────────────────────────┐
│           AURORA GLOBAL DATABASE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Cross-region replication with <1 second lag.             │
│ Low-latency global reads and disaster recovery.                │
│                                                                   │
│ Architecture:                                                   │
│ ┌─────────────────────┐         ┌─────────────────────┐       │
│ │ Primary Region      │         │ Secondary Region    │       │
│ │ (us-east-1)         │  <1s    │ (eu-west-1)         │       │
│ │ ┌───────┐ ┌───────┐ │ ──────► │ ┌───────┐ ┌───────┐ │       │
│ │ │Writer │ │Reader │ │  lag    │ │Reader │ │Reader │ │       │
│ │ └───────┘ └───────┘ │         │ └───────┘ └───────┘ │       │
│ │     Shared Storage   │         │     Shared Storage   │       │
│ └─────────────────────┘         └─────────────────────┘       │
│                                                                   │
│ Configuration:                                                  │
│ Secondary Region: [EU (Ireland) eu-west-1 ▼]                  │
│ Global cluster identifier: [prod-aurora-global]               │
│ DB instance class: [db.r6g.large ▼]                           │
│ Number of reader instances: [1]                                │
│                                                                   │
│ Key features:                                                   │
│ ├── Replication lag: <1 second (storage-level replication)   │
│ ├── Up to 5 secondary regions                                 │
│ ├── Up to 16 readers per secondary region                    │
│ ├── Planned failover: Promote secondary to primary (RPO=0)  │
│ ├── Unplanned failover: RPO <1s, RTO <1 min                │
│ ├── Write forwarding: Readers can forward writes to primary  │
│ └── ⚡ Best cross-region replication for Aurora              │
│                                                                   │
│ Use cases:                                                      │
│ ├── Disaster recovery (multi-region)                          │
│ ├── Low-latency global reads (serve users from nearest)     │
│ └── Migration between regions                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Aurora Cloning & Backtrack

```
┌─────────────────────────────────────────────────────────────────────┐
│           AURORA CLONING                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Create an instant copy of your Aurora cluster using         │
│ copy-on-write. No data is actually copied until modified!         │
│                                                                       │
│ Console: RDS → Databases → Select cluster → Actions → Create clone│
│ Clone identifier: [prod-aurora-clone-staging]                      │
│                                                                       │
│ How it works:                                                        │
│ 1. Clone is created in seconds (regardless of data size)         │
│ 2. Clone initially shares storage with source (copy-on-write)   │
│ 3. Only modified pages are duplicated (saves space)               │
│ 4. Clone is a fully independent cluster                           │
│ 5. Changes to clone don't affect source (and vice versa)         │
│                                                                       │
│ Use cases:                                                           │
│ ├── Dev/test: Clone production for realistic testing             │
│ ├── Debugging: Clone to reproduce and debug issues              │
│ ├── Analytics: Run heavy queries without affecting production    │
│ └── Schema migrations: Test migrations before applying to prod  │
│                                                                       │
│ ⚡ vs Snapshot restore:                                               │
│ ├── Clone: Seconds (any data size). Space-efficient.            │
│ └── Snapshot: Minutes to hours. Full copy. More storage cost.    │
│                                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│           AURORA BACKTRACK (MySQL only)                               │
│                                                                       │
│ What: "Rewind" your database to a specific point in time         │
│ without creating a new cluster. In-place recovery!                │
│                                                                       │
│ Console: RDS → Databases → Select cluster → Actions → Backtrack  │
│ Backtrack to: [2024-01-15 14:30:00 UTC]                           │
│                                                                       │
│ Key points:                                                          │
│ ├── Must be enabled at cluster creation                           │
│ ├── Target window: Up to 72 hours                                │
│ ├── In-place: Same cluster, same endpoint                        │
│ ├── Seconds to complete (vs minutes for PITR)                   │
│ ├── ⚠️ Affects all DB users (cluster is rewound)                 │
│ ├── Additional cost: $0.012 per million change records          │
│ └── ⚡ Perfect for: accidental DELETE, bad migration, testing    │
│                                                                       │
│ vs Point-in-Time Recovery:                                          │
│ ├── Backtrack: In-place, seconds, same endpoint                 │
│ └── PITR: New cluster, minutes, new endpoint                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Aurora Machine Learning & Integrations

```
┌─────────────────────────────────────────────────────────────────────┐
│           AURORA INTEGRATIONS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Aurora ML:                                                           │
│ ├── Call SageMaker/Comprehend directly from SQL queries          │
│ ├── No data movement needed                                       │
│ ├── Example: SELECT aws_comprehend_detect_sentiment(review_text) │
│ └── Use cases: Sentiment analysis, fraud detection in SQL        │
│                                                                       │
│ Lambda Integration:                                                  │
│ ├── Invoke Lambda functions from Aurora stored procedures        │
│ ├── Use case: Trigger notifications on data changes             │
│ └── mysql_lambda_async() / aws_lambda.invoke()                   │
│                                                                       │
│ S3 Integration:                                                      │
│ ├── LOAD DATA FROM S3: Bulk import from S3 files                │
│ ├── SELECT INTO S3: Export query results to S3                  │
│ └── Use cases: ETL, data import/export                           │
│                                                                       │
│ Zero-ETL to Redshift:                                               │
│ ├── Near real-time replication from Aurora to Redshift           │
│ ├── No ETL pipeline needed                                        │
│ ├── Analyze transactional data with Redshift analytics          │
│ └── ⚡ Simplifies analytics architecture                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & CLI Examples

```hcl
# Aurora MySQL Cluster
resource "aws_rds_cluster" "aurora" {
  cluster_identifier     = "prod-aurora-mysql"
  engine                 = "aurora-mysql"
  engine_version         = "8.0.mysql_aurora.3.06.0"
  master_username        = "admin"
  manage_master_user_password = true  # Secrets Manager
  database_name          = "myapp"

  db_subnet_group_name   = aws_db_subnet_group.aurora.name
  vpc_security_group_ids = [aws_security_group.aurora.id]

  storage_encrypted = true
  storage_type      = "aurora"  # or "aurora-iopt1" for I/O-Optimized

  backup_retention_period = 14
  preferred_backup_window = "03:00-04:00"
  deletion_protection     = true
  skip_final_snapshot     = false

  backtrack_window = 86400  # 24 hours (in seconds)

  tags = { Environment = "prod" }
}

# Writer instance
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "prod-aurora-writer"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.aurora.engine

  performance_insights_enabled = true
}

# Reader instance (Serverless v2)
resource "aws_rds_cluster_instance" "reader" {
  identifier         = "prod-aurora-reader"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.aurora.engine
}

resource "aws_rds_cluster_instance" "reader_scaling" {
  # Serverless v2 scaling config
  depends_on = [aws_rds_cluster.aurora]
}
```

```bash
# Create Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora-mysql \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.06.0 \
  --master-username admin \
  --manage-master-user-password \
  --db-subnet-group-name aurora-private \
  --vpc-security-group-ids sg-aurora \
  --storage-encrypted \
  --backup-retention-period 14 \
  --backtrack-window 86400

# Add writer instance
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-writer \
  --db-cluster-identifier prod-aurora-mysql \
  --engine aurora-mysql \
  --db-instance-class db.r6g.large

# Clone cluster
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier prod-aurora-mysql \
  --db-cluster-identifier staging-clone \
  --restore-type copy-on-write \
  --use-latest-restorable-time

# Backtrack
aws rds backtrack-db-cluster \
  --db-cluster-identifier prod-aurora-mysql \
  --backtrack-to "2024-01-15T14:30:00Z"
```

---

## Part 8: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD AURORA PATTERNS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Production SaaS Application                              │
│ ├── Aurora MySQL, I/O-Optimized                                  │
│ ├── Writer: db.r6g.xlarge (provisioned)                          │
│ ├── Reader 1: db.r6g.large (provisioned — production reads)    │
│ ├── Reader 2: Serverless v2 (2-32 ACU — peak traffic)          │
│ ├── Global Database to eu-west-1 for DR                          │
│ ├── Secrets Manager + auto-rotation                              │
│ └── Backtrack: 24-hour window                                    │
│                                                                       │
│ Pattern 2: Dev/Test with Cloning                                    │
│ ├── Clone production Aurora cluster                               │
│ ├── Staging clone: Run integration tests                         │
│ ├── Dev clones: Per-developer or per-feature-branch             │
│ ├── Serverless v2 (0.5-4 ACU) for dev clones                   │
│ └── Delete clones when done (seconds to create, cheap)          │
│                                                                       │
│ Pattern 3: Aurora + Lambda (Serverless App)                         │
│ ├── Aurora Serverless v2 (scales with traffic)                  │
│ ├── RDS Proxy (manages Lambda connection pooling)               │
│ ├── API Gateway → Lambda → RDS Proxy → Aurora                  │
│ └── Pay-per-use compute at both Lambda and Aurora level         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Aurora Quick Reference:
├── Compatibility: MySQL 5.7/8.0, PostgreSQL 13-16
├── Storage: 6 copies across 3 AZs, auto-grows to 128 TB
├── Performance: 5x MySQL, 3x PostgreSQL throughput
├── Read Replicas: Up to 15 (same shared storage)
├── Failover: <30 seconds (automatic)
├── Endpoints: Writer, Reader (load-balanced), Instance, Custom
├── Serverless v2: 0.5-256 ACU, scales in seconds
├── Global Database: <1s cross-region replication, up to 5 regions
├── Cloning: Instant (copy-on-write), independent cluster
├── Backtrack: In-place rewind up to 72 hours (MySQL only)
├── I/O-Optimized: No I/O charges when I/O > 25% of cost
└── ⚡ Choose Aurora over RDS MySQL/PG for production workloads
```

---

## What's Next?

In **Chapter 27: DynamoDB**, we'll cover AWS's fully managed NoSQL database — tables, partition/sort keys, indexes, capacity modes, streams, DAX, and full portal walkthrough.
