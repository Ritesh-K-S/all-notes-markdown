# Chapter 54: Amazon Redshift

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Redshift Fundamentals](#part-1-redshift-fundamentals)
- [Part 2: Creating a Redshift Cluster (Full Portal Walkthrough)](#part-2-creating-a-redshift-cluster-full-portal-walkthrough)
- [Part 3: Redshift Serverless](#part-3-redshift-serverless)
- [Part 4: Loading Data & Spectrum](#part-4-loading-data--spectrum)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Redshift? Data Warehouse vs Regular Database

Your regular database (RDS/Aurora) is optimized for **running your application** — quick reads and writes for individual orders, user logins, etc. This is called **OLTP** (Online Transaction Processing).

But when you want to ask questions like: "What were our total sales by region for the past 3 years across 500 million records?" — your regular database will struggle. This is **OLAP** (Online Analytical Processing), and it needs a **data warehouse**.

**Think of it this way:**
- **RDS/Aurora (OLTP)** = A cash register — fast for processing individual transactions
- **Redshift (OLAP)** = An accounting department — designed to analyze millions of transactions at once

**What makes Redshift fast for analytics?**
- **Columnar storage**: Instead of reading entire rows, Redshift reads only the columns you need (e.g., only the "sales" column from billions of rows)
- **MPP (Massively Parallel Processing)**: Query work is split across many nodes that process in parallel
- **Compression**: Columnar data compresses up to 10x better, reducing I/O

Amazon Redshift is a fully managed, petabyte-scale cloud data warehouse. It uses columnar storage and massively parallel processing (MPP) to deliver fast query performance on large datasets.

```
What you'll learn:
├── Redshift fundamentals (MPP, columnar storage)
├── Creating a Redshift cluster (portal walkthrough)
├── Redshift Serverless
├── Data loading (COPY, Spectrum, federated queries)
├── Performance tuning
└── Terraform & CLI examples
```

---

## Part 1: Redshift Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW REDSHIFT WORKS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┐                                                    │
│ │ Leader Node  │← Receives queries, plans execution              │
│ │              │← Returns results to client                      │
│ └──────┬───────┘                                                    │
│        │                                                             │
│ ┌──────┴───────┐  ┌──────────────┐  ┌──────────────┐             │
│ │ Compute Node │  │ Compute Node │  │ Compute Node │             │
│ │ (slices)     │  │ (slices)     │  │ (slices)     │             │
│ │              │  │              │  │              │             │
│ │ Columnar     │  │ Columnar     │  │ Columnar     │             │
│ │ storage      │  │ storage      │  │ storage      │             │
│ └──────────────┘  └──────────────┘  └──────────────┘             │
│                                                                       │
│ Key architecture:                                                    │
│ ├── Leader node: SQL parsing, query planning, coordination      │
│ ├── Compute nodes: Execute queries in parallel (MPP)            │
│ ├── Slices: Each node divided into slices (parallel workers)   │
│ ├── Columnar storage: Stores data by column (not row)          │
│ ├── Compression: Automatic column-level encoding                │
│ └── Result caching: Reuses results for identical queries       │
│                                                                       │
│ Node types:                                                          │
│ ├── RA3 (⚡ recommended): Managed storage (S3-backed)          │
│ │   ├── ra3.xlplus: 4 vCPU, 32 GB RAM, 32 TB managed storage │
│ │   ├── ra3.4xlarge: 12 vCPU, 96 GB RAM, 128 TB              │
│ │   └── ra3.16xlarge: 48 vCPU, 384 GB RAM, 128 TB            │
│ │   → Data stored in S3, hot data cached on local SSD         │
│ │   → Scale compute and storage independently                 │
│ ├── DC2: Dense compute (local SSD, fixed storage)              │
│ │   → dc2.large: 2 vCPU, 160 GB SSD, $0.25/hr               │
│ │   → Good for < 1 TB datasets                                │
│ └── DS2: Dense storage (legacy, use RA3 instead)               │
│                                                                       │
│ Redshift vs Athena:                                                  │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ Redshift      │ Athena               │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Type                 │ Data warehouse│ Query engine          │   │
│ │ Data storage         │ Managed/local │ S3 only              │   │
│ │ Performance          │ Fastest       │ Good                  │   │
│ │ Concurrency          │ High (hundreds│ Good (limited)       │   │
│ │                      │ of users)     │                      │   │
│ │ Cost model           │ Per node/hour │ Per TB scanned       │   │
│ │ Use case             │ BI dashboards,│ Ad-hoc queries,      │   │
│ │                      │ complex joins │ log analysis          │   │
│ │ ⚡ Best for          │ Frequent,     │ Infrequent,          │   │
│ │                      │ complex queries│ simple queries      │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Redshift Cluster (Full Portal Walkthrough)

```
Console → Redshift → Clusters → Create cluster

┌─────────────────────────────────────────────────────────────────┐
│           CLUSTER CONFIGURATION                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Cluster identifier: [analytics-warehouse]                     │
│                                                                   │
│ Plan:                                                          │
│ ○ Free trial (dc2.large, 750 hours/month for 2 months)      │
│ ● Production                                                  │
│                                                                   │
│ Node type: [ra3.xlplus ▼]                                    │
│ → ⚡ RA3 recommended for new clusters                       │
│                                                                   │
│ Number of nodes: [2] (minimum 2 for Multi-AZ)               │
│ → More nodes = more compute + storage                       │
│ → Total storage: 2 × 32 TB = 64 TB managed storage        │
│                                                                   │
│ Admin user credentials:                                       │
│ Admin user name: [admin]                                      │
│ Admin user password: ● Auto generate ○ Manual                │
│ → ⚡ Store password in Secrets Manager                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           NETWORK AND SECURITY                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ VPC: [analytics-vpc ▼]                                        │
│ Subnet group: [redshift-subnet-group ▼]                      │
│ → ⚡ Use private subnets                                     │
│                                                                   │
│ VPC security groups: [redshift-sg ▼]                         │
│ → Allow inbound on port 5439 from BI tools/app servers     │
│                                                                   │
│ Publicly accessible: ○ No (⚡ recommended)  ○ Yes            │
│ → Use VPN or bastion host for direct access                 │
│                                                                   │
│ Availability Zone: [us-east-1a ▼]                            │
│ Multi-AZ: ☑ Enable (RA3 only)                               │
│ → ⚡ Automatic failover to standby in another AZ           │
│                                                                   │
│ Encryption:                                                    │
│ ☑ Use AWS Key Management Service (KMS)                      │
│ KMS key: [aws/redshift ▼]                                   │
│                                                                   │
│ Enhanced VPC routing: ☑ Enable                               │
│ → Forces COPY/UNLOAD through VPC (not public internet)    │
│ → ⚡ Required for compliance (data doesn't traverse internet)│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           ADDITIONAL SETTINGS                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Database configuration:                                        │
│ Database name: [analytics]                                    │
│ Port: [5439] (default)                                       │
│                                                                   │
│ Maintenance:                                                   │
│ Maintenance window: [Sun 02:00 - Sun 02:30 UTC]             │
│ ☑ Allow version upgrade                                      │
│                                                                   │
│ Backup:                                                        │
│ Automated snapshot retention: [7] days (1-35)                │
│ ☑ Enable cross-region snapshot                               │
│ Destination region: [us-west-2 ▼]                            │
│ Snapshot copy retention: [7] days                            │
│                                                                   │
│ Monitoring:                                                    │
│ ☑ Enable audit logging                                       │
│ S3 bucket: [redshift-audit-logs ▼]                           │
│                                                                   │
│ IAM roles (for COPY/UNLOAD/Spectrum):                       │
│ [RedshiftS3ReadRole ▼]                                       │
│ → Allows Redshift to read from S3 (COPY command)           │
│ → Allows Redshift Spectrum queries on S3                   │
│                                                                   │
│                    [Create cluster]                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Redshift Serverless

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDSHIFT SERVERLESS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Redshift → Serverless dashboard → Create workgroup       │
│                                                                       │
│ Workgroup name: [analytics-serverless]                              │
│                                                                       │
│ Base capacity: [32] RPUs (Redshift Processing Units)               │
│ → Range: 8 to 512 RPUs                                            │
│ → ⚡ Start with 32, auto-scales based on workload                │
│                                                                       │
│ Namespace: [analytics-namespace]                                    │
│ → Logical grouping: databases, schemas, users                     │
│                                                                       │
│ Admin credentials:                                                   │
│ ● Manage admin credentials in Secrets Manager                     │
│ ○ Customize admin user                                             │
│                                                                       │
│ VPC: [analytics-vpc ▼]                                              │
│ Subnets: ☑ private-1  ☑ private-2  ☑ private-3                  │
│ Security groups: [redshift-serverless-sg ▼]                        │
│ Publicly accessible: ○ No (⚡ recommended)                        │
│                                                                       │
│ Encryption: ☑ AWS KMS                                               │
│ Enhanced VPC routing: ☑ Enable                                     │
│                                                                       │
│ Serverless vs Provisioned:                                          │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ Provisioned   │ Serverless           │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Management           │ You size nodes│ Auto-scales          │   │
│ │ Scaling              │ Manual resize │ Automatic            │   │
│ │ Cost model           │ Per node/hour │ Per RPU-hour (used)  │   │
│ │ Idle cost            │ Still charged │ Scales to 0 *        │   │
│ │ Best for             │ Predictable   │ Variable/intermittent│   │
│ │                      │ workloads     │ workloads            │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│ * Serverless charges minimum when actively querying               │
│                                                                       │
│ ⚡ Use Serverless for: Development, intermittent analytics,        │
│   unpredictable workloads. Use Provisioned for: Steady, high-    │
│   concurrency production dashboards.                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Loading Data & Spectrum

```
┌─────────────────────────────────────────────────────────────────────┐
│           LOADING DATA                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: COPY command (⚡ fastest for bulk loads)                  │
│ COPY orders FROM 's3://data-bucket/orders/'                        │
│ IAM_ROLE 'arn:aws:iam::123456:role/RedshiftS3ReadRole'            │
│ FORMAT AS PARQUET;                                                   │
│                                                                       │
│ -- CSV with options                                                  │
│ COPY orders FROM 's3://data-bucket/orders.csv'                     │
│ IAM_ROLE 'arn:aws:iam::123456:role/RedshiftS3ReadRole'            │
│ CSV                                                                   │
│ IGNOREHEADER 1                                                       │
│ GZIP                                                                  │
│ REGION 'us-east-1';                                                  │
│                                                                       │
│ ⚡ COPY best practices:                                              │
│ ├── Split files into multiples of slice count                   │
│ ├── Use compressed files (GZIP, LZO, BZIP2)                   │
│ ├── Use Parquet/ORC for best performance                       │
│ └── Use manifest file for exact file control                   │
│                                                                       │
│ Method 2: INSERT (for small amounts)                                │
│ INSERT INTO orders VALUES (1, 'Widget', 9.99);                     │
│                                                                       │
│ Method 3: Firehose → Redshift (streaming)                          │
│ Kinesis Data Firehose → S3 → COPY → Redshift                     │
│ → Near real-time data loading                                     │
│                                                                       │
│ UNLOAD (export data):                                               │
│ UNLOAD ('SELECT * FROM orders WHERE year=2024')                    │
│ TO 's3://data-bucket/export/orders_2024_'                          │
│ IAM_ROLE 'arn:aws:iam::123456:role/RedshiftS3ReadRole'            │
│ PARQUET;                                                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           REDSHIFT SPECTRUM                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Query S3 data directly from Redshift (no loading needed)    │
│                                                                       │
│ -- Create external schema (link to Glue Data Catalog)             │
│ CREATE EXTERNAL SCHEMA spectrum_schema                              │
│ FROM DATA CATALOG                                                    │
│ DATABASE 'analytics'                                                │
│ IAM_ROLE 'arn:aws:iam::123456:role/RedshiftSpectrumRole';        │
│                                                                       │
│ -- Query S3 data via Spectrum                                       │
│ SELECT s.page, COUNT(*) as views                                   │
│ FROM spectrum_schema.clickstream s                                 │
│ WHERE s.year = '2024'                                               │
│ GROUP BY s.page;                                                     │
│                                                                       │
│ -- Join Redshift table with S3 data                                │
│ SELECT o.order_id, c.page                                           │
│ FROM orders o                                                        │
│ JOIN spectrum_schema.clickstream c                                  │
│   ON o.user_id = c.user_id;                                        │
│                                                                       │
│ ⚡ Use Spectrum for:                                                  │
│ ├── Historical/cold data (keep in S3, query on-demand)          │
│ ├── Joining hot (Redshift) + cold (S3) data                    │
│ └── Avoid loading infrequently queried data                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# Redshift cluster
resource "aws_redshift_cluster" "analytics" {
  cluster_identifier  = "analytics-warehouse"
  database_name       = "analytics"
  master_username     = "admin"
  master_password     = var.redshift_password
  node_type           = "ra3.xlplus"
  number_of_nodes     = 2
  port                = 5439

  vpc_security_group_ids = [aws_security_group.redshift.id]
  cluster_subnet_group_name = aws_redshift_subnet_group.main.name
  publicly_accessible    = false
  encrypted              = true
  kms_key_id             = aws_kms_key.redshift.arn
  enhanced_vpc_routing   = true

  iam_roles = [aws_iam_role.redshift_s3.arn]

  automated_snapshot_retention_period = 7
  skip_final_snapshot                 = false
  final_snapshot_identifier           = "analytics-final-snapshot"
}

resource "aws_redshift_subnet_group" "main" {
  name       = "redshift-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

# Redshift Serverless
resource "aws_redshiftserverless_namespace" "analytics" {
  namespace_name      = "analytics-namespace"
  db_name             = "analytics"
  admin_username      = "admin"
  admin_user_password = var.redshift_password
  kms_key_id          = aws_kms_key.redshift.arn
  iam_roles           = [aws_iam_role.redshift_s3.arn]
}

resource "aws_redshiftserverless_workgroup" "analytics" {
  namespace_name = aws_redshiftserverless_namespace.analytics.namespace_name
  workgroup_name = "analytics-serverless"
  base_capacity  = 32

  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.redshift.id]
  publicly_accessible = false
  enhanced_vpc_routing = true
}
```

```bash
# Create cluster
aws redshift create-cluster \
  --cluster-identifier analytics-warehouse \
  --node-type ra3.xlplus \
  --number-of-nodes 2 \
  --master-username admin \
  --master-user-password 'SecurePassword123!' \
  --db-name analytics

# Resize cluster
aws redshift resize-cluster \
  --cluster-identifier analytics-warehouse \
  --node-type ra3.4xlarge \
  --number-of-nodes 4

# Create snapshot
aws redshift create-cluster-snapshot \
  --cluster-identifier analytics-warehouse \
  --snapshot-identifier analytics-backup-20240115

# Pause cluster (stop billing for compute)
aws redshift pause-cluster --cluster-identifier analytics-warehouse

# Resume cluster
aws redshift resume-cluster --cluster-identifier analytics-warehouse
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Enterprise Data Warehouse                                │
│ Source Systems → Glue ETL → S3 (staging) → COPY → Redshift      │
│                                                   → QuickSight    │
│ ├── Nightly ETL from RDS, DynamoDB, APIs                        │
│ ├── Star schema (fact + dimension tables)                       │
│ ├── Distribution keys on join columns                           │
│ ├── Sort keys on filter/order-by columns                       │
│ └── BI dashboards via QuickSight                                │
│                                                                       │
│ Pattern 2: Data Lakehouse (Redshift + Spectrum)                    │
│ Hot data: Redshift tables (frequent queries)                      │
│ Cold data: S3 via Spectrum (infrequent queries)                   │
│ ├── Recent 3 months in Redshift (fast)                          │
│ ├── Historical data in S3 Parquet (cheap)                       │
│ ├── JOIN across hot + cold data seamlessly                     │
│ └── UNLOAD old data from Redshift → S3 periodically           │
│                                                                       │
│ Pattern 3: Real-Time + Batch Analytics                              │
│ Kinesis → Firehose → Redshift (near real-time)                   │
│ Glue ETL → S3 → COPY → Redshift (nightly batch)                │
│ → Combined view of streaming + batch data                       │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use RA3 nodes (separate compute + storage)                   │
│ 2. Choose distribution key wisely (JOIN columns)               │
│ 3. Use sort keys for frequently filtered columns               │
│ 4. COPY with Parquet for fastest loading                        │
│ 5. Enable Enhanced VPC routing for security                     │
│ 6. Use Spectrum for cold/historical data in S3                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Redshift Quick Reference:
├── Type: Columnar MPP data warehouse (petabyte-scale)
├── Node types: RA3 (⚡ recommended), DC2, DS2 (legacy)
├── Port: 5439 (PostgreSQL-compatible)
├── Loading: COPY from S3 (⚡ fastest), INSERT, Firehose
├── Spectrum: Query S3 data directly (Glue Catalog)
├── Serverless: Auto-scaling, pay per RPU-hour
├── Encryption: KMS, at-rest and in-transit
├── Enhanced VPC routing: Force S3 traffic through VPC
├── Backups: Automated snapshots (1-35 days retention)
├── Multi-AZ: Available on RA3 (automatic failover)
├── Distribution: KEY (join column), EVEN, ALL
├── Sort keys: Compound or interleaved
├── ⚡ RA3 + Spectrum = data lakehouse architecture
├── ⚡ COPY with Parquet = fastest bulk loading
└── ⚡ Pause/resume cluster to save costs during off-hours
```

---

## What's Next?

In **Chapter 55: Amazon SageMaker**, we'll cover the ML platform, notebooks, model training, deployment, and MLOps for building and deploying machine learning models.
