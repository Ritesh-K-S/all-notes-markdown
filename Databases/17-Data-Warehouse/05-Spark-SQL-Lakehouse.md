# 6.5 — Apache Spark SQL & Data Lakehouse 🔴🔥

> **"If the data warehouse was the library, and the data lake was the ocean, the Lakehouse is a library built on the ocean — organized knowledge with unlimited capacity."**

> **Level:** 🔴 Advanced | 🔥 High Demand  
> **Time to Master:** ~5-6 hours  
> **Prerequisites:** Chapter 6.1 (Data Warehouse Concepts), familiarity with distributed systems

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **Apache Spark's architecture** and why it dominates big data processing
- Write powerful analytics with **Spark SQL** — the SQL engine for massive data
- Master the **Lakehouse paradigm** — why it's replacing both lakes and warehouses
- Compare **Delta Lake**, **Apache Iceberg**, and **Apache Hudi** — the three lakehouse formats
- Know when to use **Databricks** vs **open-source Spark**
- Understand how **ACID transactions** came to data lakes — and changed everything

---

## 🧠 The Evolution — How We Got Here

```
                    THE DATA PLATFORM TIMELINE

2000s: DATA WAREHOUSE ERA
┌──────────────────────────────────────────────────────────────┐
│  • Structured data only (tables, SQL)                        │
│  • Expensive (Teradata, Oracle, Netezza: $$$$$)              │
│  • Limited to structured business data                       │
│  • ETL into rigid schemas                                    │
│  • ✅ Reliable, ACID, fast queries                           │
│  • ❌ Expensive, inflexible, structured only                 │
└──────────────────────────────────────────────────────────────┘
                            ↓ "We need to store EVERYTHING cheaply"
2010s: DATA LAKE ERA  
┌──────────────────────────────────────────────────────────────┐
│  • Hadoop + HDFS + Hive                                      │
│  • Store ANY data type (structured, semi, unstructured)      │
│  • Very cheap storage (commodity hardware)                    │
│  • Schema-on-read (decide structure when querying)           │
│  • ✅ Cheap, flexible, any format                            │
│  • ❌ Unreliable ("data swamp"), no ACID, slow queries       │
│  • ❌ No UPDATE/DELETE, no time travel, no quality guarantees│
└──────────────────────────────────────────────────────────────┘
                            ↓ "We need the best of BOTH"
2020s: DATA LAKEHOUSE ERA
┌──────────────────────────────────────────────────────────────┐
│  • Spark + Delta Lake / Iceberg / Hudi                       │
│  • Cheap storage (S3/ADLS/GCS) + warehouse reliability       │
│  • ACID transactions on data lakes!                          │
│  • Schema enforcement + evolution                            │
│  • Time travel, UPDATE/DELETE, MERGE                         │
│  • ✅ Cheap, flexible, reliable, fast, ACID                  │
│  • ✅ One copy of data for ALL workloads (BI + ML + ELT)     │
│  • 🔥 The current state of the art                           │
└──────────────────────────────────────────────────────────────┘
```

### Why Data Lakes Failed (and Lakehouses Succeeded)

```
THE DATA SWAMP PROBLEM:

Data Lake started great:
  Year 1: "Let's store everything in S3! It's so cheap!"
  Year 2: "We have petabytes of data! We're data-driven!"
  Year 3: "Wait... where's the customer data? Which version is current?"
  Year 4: "Someone wrote corrupt Parquet files. Our ML pipeline is broken."
  Year 5: "We basically have a SWAMP. Nobody trusts this data."

Root causes:
  ❌ No schema enforcement → corrupt data enters undetected
  ❌ No ACID transactions → partial writes = corrupt files
  ❌ No UPDATE/DELETE → can't fix mistakes or comply with GDPR
  ❌ No versioning → "which file is the latest?"
  ❌ No quality guarantees → garbage in, garbage out

Lakehouse fixes ALL of this:
  ✅ Schema enforcement → bad data rejected at write time
  ✅ ACID transactions → atomic writes, consistent reads
  ✅ UPDATE/DELETE/MERGE → fix data, comply with regulations
  ✅ Time travel → version history, rollback mistakes
  ✅ Audit log → who changed what, when
```

---

## ⚡ Apache Spark — The Engine Behind It All

### What is Apache Spark?

```
Apache Spark = Distributed computing engine for large-scale data processing

Key Facts:
┌──────────────────────────────────────────────────────────────┐
│ • Created at UC Berkeley (2009), donated to Apache (2013)    │
│ • 100x faster than Hadoop MapReduce (in-memory processing)  │
│ • Processes petabytes of data across thousands of machines   │
│ • APIs: SQL, Python (PySpark), Scala, Java, R               │
│ • Unified engine: batch, streaming, ML, graph — one platform │
│ • Used by: Netflix, Uber, Apple, NASA, CERN                  │
│ • Most active open-source big data project                   │
└──────────────────────────────────────────────────────────────┘
```

### Spark Architecture

```
                    ┌─────────────────────────┐
                    │     DRIVER PROGRAM       │
                    │   (Your application)      │
                    │                          │
                    │  SparkSession / Context   │
                    │  • Parses your code       │
                    │  • Creates execution plan │
                    │  • Coordinates workers    │
                    └───────────┬──────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │     CLUSTER MANAGER       │
                    │  (YARN / Kubernetes /     │
                    │   Standalone / Mesos)     │
                    │  • Allocates resources    │
                    │  • Manages worker nodes   │
                    └───────────┬──────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │  EXECUTOR 1  │  │  EXECUTOR 2  │  │  EXECUTOR 3  │
     │  (Worker)    │  │  (Worker)    │  │  (Worker)    │
     │              │  │              │  │              │
     │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
     │  │ Task 1 │  │  │  │ Task 4 │  │  │  │ Task 7 │  │
     │  │ Task 2 │  │  │  │ Task 5 │  │  │  │ Task 8 │  │
     │  │ Task 3 │  │  │  │ Task 6 │  │  │  │ Task 9 │  │
     │  └────────┘  │  │  └────────┘  │  │  └────────┘  │
     │   Cache/RAM  │  │   Cache/RAM  │  │   Cache/RAM  │
     └──────────────┘  └──────────────┘  └──────────────┘
            │                 │                  │
     ┌──────▼──────┐  ┌──────▼──────┐  ┌────────▼────┐
     │ Data Part 1 │  │ Data Part 2 │  │ Data Part 3 │
     │ (S3/HDFS)   │  │ (S3/HDFS)   │  │ (S3/HDFS)   │
     └─────────────┘  └─────────────┘  └─────────────┘
```

### Why Spark is Fast

```
HADOOP MAPREDUCE (2006):
  Read from disk → Process → Write to disk → Read from disk → Process → Write...
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ DISK │───►│ MAP  │───►│ DISK │───►│REDUCE│───►│ DISK │
  └──────┘    └──────┘    └──────┘    └──────┘    └──────┘
  Every step hits disk = SLOW (disk I/O is 100x slower than RAM)

APACHE SPARK (2014):
  Read from disk → Process IN MEMORY → Continue in memory → Write once
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ DISK │───►│  RAM │───►│  RAM │───►│  RAM │───►│ DISK │
  └──────┘    └──────┘    └──────┘    └──────┘    └──────┘
  Intermediate results stay in memory = FAST (100x faster)

  Plus:
  • Lazy evaluation (optimizes before executing)
  • Catalyst optimizer (SQL-like query planning)
  • Tungsten engine (memory management, code generation)
  • Adaptive Query Execution (runtime optimization)
```

---

## 🔷 Spark SQL — SQL on Steroids

Spark SQL lets you run **standard SQL** on massive datasets distributed across a cluster.

### Getting Started

```sql
-- Spark SQL works just like regular SQL, but on distributed data

-- Create a SparkSession (PySpark)
-- from pyspark.sql import SparkSession
-- spark = SparkSession.builder.appName("Analytics").getOrCreate()

-- Read data from various sources
CREATE TEMPORARY VIEW sales
USING parquet
OPTIONS (path 's3://data-lake/sales/*.parquet');

CREATE TEMPORARY VIEW customers
USING csv
OPTIONS (path 's3://data-lake/customers/*.csv', header 'true', inferSchema 'true');

-- Now use standard SQL!
SELECT 
    c.region,
    DATE_TRUNC('month', s.order_date) AS month,
    SUM(s.amount) AS revenue,
    COUNT(DISTINCT s.customer_id) AS unique_customers,
    SUM(s.amount) / COUNT(DISTINCT s.customer_id) AS revenue_per_customer
FROM sales s
JOIN customers c ON s.customer_id = c.customer_id
WHERE s.order_date >= '2025-01-01'
GROUP BY c.region, DATE_TRUNC('month', s.order_date)
ORDER BY month, revenue DESC;

-- This query might scan 500 GB across 1,000 Parquet files
-- Spark distributes it across 100 executors
-- Completes in 30 seconds instead of hours! ⚡
```

### Spark SQL vs Traditional SQL

```
┌──────────────────┬──────────────────────┬────────────────────────┐
│ Feature          │ Traditional SQL DB   │ Spark SQL              │
├──────────────────┼──────────────────────┼────────────────────────┤
│ Data size        │ GB to low TB         │ TB to PB               │
│ Data location    │ Database storage     │ S3, HDFS, JDBC, etc.   │
│ Processing       │ Single machine       │ Distributed cluster    │
│ Data formats     │ Internal format      │ Parquet, ORC, CSV,     │
│                  │                      │ JSON, Avro, Delta, etc.│
│ Schema           │ Schema-on-write      │ Schema-on-read + write │
│ Latency          │ Milliseconds         │ Seconds (batch)        │
│ Best for         │ Transactions, OLTP   │ Analytics, ETL, ML     │
│ UPDATE/DELETE    │ Native               │ Via Lakehouse formats  │
│ Indexes          │ B-Tree, Hash, etc.   │ None (partition pruning│
│                  │                      │ + column pruning)      │
└──────────────────┴──────────────────────┴────────────────────────┘
```

### Spark SQL Power Features

```sql
-- Window Functions (same as traditional SQL — works on massive data)
SELECT 
    product_id,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY product_id 
        ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    LAG(amount) OVER (PARTITION BY product_id ORDER BY order_date) AS prev_amount,
    amount - LAG(amount) OVER (PARTITION BY product_id ORDER BY order_date) AS change
FROM sales;

-- CTEs work perfectly
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS revenue
    FROM sales
    GROUP BY 1
),
with_growth AS (
    SELECT 
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
        (revenue - LAG(revenue) OVER (ORDER BY month)) / 
            LAG(revenue) OVER (ORDER BY month) * 100 AS growth_pct
    FROM monthly_revenue
)
SELECT * FROM with_growth ORDER BY month;

-- CUBE and ROLLUP for multi-level aggregations
SELECT 
    COALESCE(region, 'ALL REGIONS') AS region,
    COALESCE(category, 'ALL CATEGORIES') AS category,
    SUM(amount) AS revenue
FROM sales
GROUP BY CUBE(region, category)
ORDER BY region, category;
```

---

## 🏠 The Lakehouse — Architecture Deep Dive

### What is a Lakehouse?

```
LAKEHOUSE = Data Lake (cheap storage) + Data Warehouse (reliability + performance)

┌─────────────────────────────────────────────────────────────────┐
│                     LAKEHOUSE ARCHITECTURE                       │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  📊 CONSUMPTION LAYER                                     │  │
│  │  BI Tools (Tableau, Power BI) │ ML (TensorFlow, PyTorch)  │  │
│  │  SQL Analytics │ Data Science │ Streaming │ Applications   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  ⚡ PROCESSING LAYER                                      │  │
│  │  Apache Spark │ Trino/Presto │ Flink │ Snowflake │ etc.  │  │
│  │  (compute engines that read lakehouse tables)              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  📋 TABLE FORMAT LAYER ← THE KEY INNOVATION               │  │
│  │                                                           │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐      │  │
│  │  │ DELTA LAKE  │  │ APACHE       │  │ APACHE HUDI │      │  │
│  │  │ (Databricks)│  │ ICEBERG      │  │ (Uber)      │      │  │
│  │  └─────────────┘  │ (Netflix)    │  └─────────────┘      │  │
│  │                    └──────────────┘                        │  │
│  │  These add: ACID, time travel, schema evolution,          │  │
│  │  UPDATE/DELETE/MERGE, audit logs, versioning              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  📦 STORAGE LAYER (Open, Cheap)                           │  │
│  │  Amazon S3 │ Azure ADLS │ Google GCS │ HDFS              │  │
│  │  Data stored as Parquet/ORC files + metadata logs         │  │
│  │  $0.023/GB/month (S3) — incredibly cheap                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔺 Delta Lake — The Databricks Standard

Delta Lake is the **most popular** lakehouse format, created by Databricks (the company behind Spark).

### How Delta Lake Works

```
Regular Parquet (Data Lake):
  s3://bucket/sales/
    ├── part-001.parquet
    ├── part-002.parquet
    └── part-003.parquet
  → Just files. No transactions. No versioning. No ACID.
  → Partial writes = corrupt data. No UPDATE possible.

Delta Lake:
  s3://bucket/sales/
    ├── _delta_log/                    ← TRANSACTION LOG (the secret sauce!)
    │   ├── 00000000000000000000.json  ← Version 0: initial load
    │   ├── 00000000000000000001.json  ← Version 1: INSERT 1000 rows
    │   ├── 00000000000000000002.json  ← Version 2: UPDATE 50 rows
    │   └── 00000000000000000003.json  ← Version 3: DELETE 10 rows
    ├── part-00001-xxx.parquet         ← Data files
    ├── part-00002-xxx.parquet
    ├── part-00003-xxx.parquet
    └── part-00004-xxx.parquet         ← New data from UPDATE

The _delta_log tracks:
  • Which files to read (add/remove actions)
  • Schema information
  • Transaction metadata
  • Commit timestamps
  → ACID transactions on top of Parquet files!
```

### Delta Lake Operations

```sql
-- Create a Delta table
CREATE TABLE fact_sales (
    sale_id     BIGINT,
    order_date  DATE,
    customer_id INT,
    product_id  INT,
    amount      DECIMAL(12,2),
    quantity    INT
)
USING DELTA
LOCATION 's3://data-lake/delta/fact_sales/'
PARTITIONED BY (order_date);

-- INSERT (same as normal SQL)
INSERT INTO fact_sales VALUES (1, '2026-06-02', 42, 101, 999.00, 1);

-- UPDATE (impossible on plain data lake — Delta makes it work!)
UPDATE fact_sales 
SET amount = 899.00 
WHERE sale_id = 1;

-- DELETE (GDPR compliance: "right to be forgotten")
DELETE FROM fact_sales 
WHERE customer_id = 42;  -- Customer requested data deletion

-- MERGE (upsert: update if exists, insert if new)
MERGE INTO fact_sales AS target
USING daily_updates AS source
ON target.sale_id = source.sale_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- ⭐ This is the killer feature — UPDATE/DELETE/MERGE on a data lake!
-- Traditional data lakes: "You want to update a Parquet file? LOL, rebuild everything."
-- Delta Lake: "Sure, here's a simple SQL UPDATE."
```

### Delta Lake Time Travel

```sql
-- Query version 0 (original data)
SELECT * FROM fact_sales VERSION AS OF 0;

-- Query data as of a timestamp
SELECT * FROM fact_sales TIMESTAMP AS OF '2026-06-01';

-- See all versions
DESCRIBE HISTORY fact_sales;

-- Result:
-- version | timestamp           | operation | operationParams
-- 3       | 2026-06-02 15:30:00 | DELETE    | {"predicate":"customer_id = 42"}
-- 2       | 2026-06-02 14:00:00 | UPDATE    | {"predicate":"sale_id = 1"}
-- 1       | 2026-06-02 12:00:00 | WRITE     | {"mode":"Append","partitionBy":"[order_date]"}
-- 0       | 2026-06-01 10:00:00 | WRITE     | {"mode":"Overwrite"}

-- Restore to a previous version
RESTORE TABLE fact_sales TO VERSION AS OF 1;
-- Table is now back to version 1 state!
```

### Delta Lake Optimization

```sql
-- OPTIMIZE: Compact small files into larger ones (reduces read overhead)
OPTIMIZE fact_sales;
-- Before: 10,000 small files (1 MB each) → 100 large files (100 MB each)
-- Query goes from reading 10,000 files to reading 100 → much faster!

-- Z-ORDER: Cluster data by specific columns (like Snowflake clustering)
OPTIMIZE fact_sales ZORDER BY (customer_id, product_id);
-- Co-locates related data → partition pruning on these columns

-- VACUUM: Remove old files no longer referenced by the transaction log
VACUUM fact_sales RETAIN 168 HOURS;  -- Keep 7 days of history
-- Frees storage by deleting obsolete Parquet files
-- ⚠️ After VACUUM, time travel for deleted versions is NOT possible

-- AUTO-OPTIMIZE (Databricks feature)
ALTER TABLE fact_sales SET TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.autoOptimize.autoCompact' = 'true'
);
```

---

## 🧊 Apache Iceberg — The Netflix Standard

Apache Iceberg was created at Netflix to handle **petabyte-scale** tables with full reliability.

```
Key Differences from Delta Lake:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  DELTA LAKE:                                                 │
│  • Transaction log = JSON files in _delta_log/               │
│  • Strong Spark/Databricks integration                       │
│  • Schema evolution: add/rename columns                      │
│                                                              │
│  APACHE ICEBERG:                                             │
│  • Transaction log = metadata files + manifest lists +       │
│    manifest files (3-level hierarchy)                        │
│  • Engine-agnostic (Spark, Trino, Flink, Hive, Dremio)      │
│  • Schema evolution: add/rename/drop/reorder columns         │
│  • Hidden partitioning (users don't need to know partitions) │
│  • Partition evolution (change partitioning without rewrite)  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Iceberg's Killer Feature: Hidden Partitioning

```sql
-- Traditional partitioning (Hive/Delta):
-- User MUST include partition column in queries

CREATE TABLE events (
    event_id     BIGINT,
    event_time   TIMESTAMP,
    event_type   STRING,
    user_id      BIGINT
)
PARTITIONED BY (date STRING);  -- User must know about this!

-- Query MUST use the partition column correctly:
SELECT * FROM events WHERE date = '2026-06-02';     -- ✅ Pruned
SELECT * FROM events WHERE event_time > '2026-06-02'; -- ❌ NOT pruned!
-- The user needs to know "date" is derived from "event_time" — error-prone

-- ICEBERG Hidden Partitioning:
CREATE TABLE events (
    event_id     BIGINT,
    event_time   TIMESTAMP,
    event_type   STRING,
    user_id      BIGINT
)
USING ICEBERG
PARTITIONED BY (days(event_time));  -- Partition by DAY of event_time

-- Now users can just query naturally:
SELECT * FROM events WHERE event_time > '2026-06-02';  -- ✅ PRUNED!
-- Iceberg automatically translates the filter to partition pruning
-- Users don't need to know about partitioning at all!

-- Partition transforms:
-- years(ts)     → Partition by year
-- months(ts)    → Partition by month
-- days(ts)      → Partition by day
-- hours(ts)     → Partition by hour
-- bucket(N, col)→ Hash bucket into N partitions
-- truncate(W, col)→ Truncate to width W
```

### Iceberg Partition Evolution

```sql
-- Start with monthly partitions
CREATE TABLE logs USING ICEBERG
PARTITIONED BY (months(event_time));

-- Table grows huge → switch to daily partitions WITHOUT rewriting data!
ALTER TABLE logs ADD PARTITION FIELD days(event_time);
ALTER TABLE logs DROP PARTITION FIELD months(event_time);

-- Old data: still monthly partitions (not rewritten)
-- New data: daily partitions
-- Iceberg handles the mixed partitioning transparently!

-- 💡 Delta Lake and Hive CANNOT do this — they require full data rewrite
```

---

## 🔥 Apache Hudi — The Uber Standard

Apache Hudi (Hadoop Upserts Deletes and Incrementals) was created at Uber for **near-real-time** data lakehouse workloads.

```
HUDI'S UNIQUE STRENGTHS:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. RECORD-LEVEL UPSERTS (not file-level)                   │
│     → Uber: billions of trip records, need to update status  │
│     → Hudi tracks individual records with primary keys       │
│     → Update 1 row without rewriting entire file             │
│                                                              │
│  2. INCREMENTAL QUERIES                                     │
│     → "Show me only CHANGES since last query"               │
│     → Perfect for incremental ETL pipelines                 │
│     → No need to re-scan entire table                       │
│                                                              │
│  3. TWO TABLE TYPES:                                        │
│     Copy-on-Write (CoW):                                    │
│     → Rewrites entire file on update                         │
│     → Slower writes, faster reads                            │
│     → Best for: read-heavy, batch workloads                  │
│                                                              │
│     Merge-on-Read (MoR):                                    │
│     → Writes to delta log, merges on read                    │
│     → Faster writes, slightly slower reads                   │
│     → Best for: write-heavy, near-real-time                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🆚 Delta Lake vs Iceberg vs Hudi — The Big Comparison

```
┌──────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Feature          │ Delta Lake      │ Apache Iceberg   │ Apache Hudi     │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Created by       │ Databricks      │ Netflix          │ Uber            │
│ Primary engine   │ Spark           │ Engine-agnostic  │ Spark           │
│ ACID             │ ✅              │ ✅               │ ✅              │
│ Time Travel      │ ✅              │ ✅               │ ✅              │
│ Schema Evolution │ ✅ (limited)    │ ✅ (best)        │ ✅              │
│ Part. Evolution  │ ❌ (rewrite)    │ ✅ (in-place)    │ ❌ (rewrite)    │
│ Hidden Partition │ ❌              │ ✅               │ ❌              │
│ UPDATE/DELETE    │ ✅              │ ✅               │ ✅ (best perf)  │
│ Incremental Read │ ✅ (CDF)        │ ✅ (snapshots)   │ ✅ (best)       │
│ Streaming        │ ✅              │ ✅               │ ✅ (best)       │
│ File Format      │ Parquet         │ Parquet/ORC/Avro │ Parquet         │
│ Log Format       │ JSON files      │ Avro manifests   │ Timeline + logs │
│ Multi-engine     │ 🟡 (improving)  │ ✅ (best)        │ 🟡              │
│ Ecosystem        │ Databricks      │ Broad (growing)  │ AWS EMR         │
│ Community        │ Very large      │ Rapidly growing  │ Moderate        │
│ Maturity         │ Most mature     │ Production-ready │ Production-ready│
│ Best for         │ Databricks users│ Multi-engine     │ Near-real-time  │
│                  │ Spark-heavy     │ environments     │ upsert-heavy    │
└──────────────────┴─────────────────┴─────────────────┴─────────────────┘

🔥 INDUSTRY TREND (2025-2026):
• Delta Lake: dominant in Databricks ecosystem
• Apache Iceberg: gaining fastest adoption (Snowflake, AWS, Dremio, Trino)
• Apache Hudi: strong in AWS ecosystem (EMR, Glue)
• Universal adoption: most engines now support ALL three formats
```

---

## 🧱 Databricks — The Lakehouse Platform

Databricks is the **commercial platform** built around Spark + Delta Lake.

```
┌──────────────────────────────────────────────────────────────┐
│  DATABRICKS PLATFORM                                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🔷 DATABRICKS SQL (SQL Warehouse)                    │    │
│  │  → BI / Analytics queries on lakehouse tables         │    │
│  │  → Connects to Tableau, Power BI, Looker              │    │
│  │  → Serverless SQL compute                             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  📊 DATA ENGINEERING (ETL/ELT)                        │    │
│  │  → Spark jobs for data transformation                 │    │
│  │  → Auto Loader (streaming file ingestion)             │    │
│  │  → Delta Live Tables (declarative ETL pipelines)      │    │
│  │  → Workflows (orchestration like Airflow)             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🤖 DATA SCIENCE & ML                                 │    │
│  │  → MLflow (ML experiment tracking + model registry)   │    │
│  │  → Notebooks (Python, Scala, R, SQL)                  │    │
│  │  → Feature Store                                      │    │
│  │  → AutoML                                             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🏛️ UNITY CATALOG (Governance)                       │    │
│  │  → Centralized metadata management                    │    │
│  │  → Fine-grained access control (table, column, row)   │    │
│  │  → Data lineage tracking                              │    │
│  │  → Audit logging                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  Runs on: AWS, Azure, GCP (all three!)                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 Real-World Lakehouse Pipeline

```
                    COMPLETE LAKEHOUSE PIPELINE

  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  📥 INGESTION                                               │
  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
  │  │ Kafka   │ │  APIs   │ │  JDBC   │ │  Files  │          │
  │  │ Streams │ │ (REST)  │ │ (MySQL) │ │ (CSV/   │          │
  │  │         │ │         │ │         │ │ Parquet) │          │
  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
  │       └───────────┴──────────┴────────────┘                │
  │                        │                                    │
  │  ┌─────────────────────▼─────────────────────────────────┐  │
  │  │  🥉 BRONZE LAYER (Raw / Landing)                       │  │
  │  │  • Raw data as-is from sources                         │  │
  │  │  • No transformations                                  │  │
  │  │  • Delta format (for ACID + time travel)               │  │
  │  │  • Append-only                                         │  │
  │  │  • Used for: audit trail, reprocessing, debugging      │  │
  │  │  Example: raw_orders, raw_customers, raw_events        │  │
  │  └─────────────────────┬─────────────────────────────────┘  │
  │                        │ Cleanse + Validate                 │
  │  ┌─────────────────────▼─────────────────────────────────┐  │
  │  │  🥈 SILVER LAYER (Cleaned / Conformed)                 │  │
  │  │  • Deduplicated, validated, standardized               │  │
  │  │  • Schema enforced                                     │  │
  │  │  • Business rules applied                              │  │
  │  │  • JOINs across sources (integrated view)              │  │
  │  │  • Used for: data science, ad-hoc analysis             │  │
  │  │  Example: cleaned_orders, customers_360                │  │
  │  └─────────────────────┬─────────────────────────────────┘  │
  │                        │ Aggregate + Enrich                 │
  │  ┌─────────────────────▼─────────────────────────────────┐  │
  │  │  🥇 GOLD LAYER (Business-Level / Aggregated)           │  │
  │  │  • Aggregated, business-ready tables                   │  │
  │  │  • Star schema / dimensional model                     │  │
  │  │  • Pre-computed KPIs and metrics                       │  │
  │  │  • Optimized for BI dashboards                         │  │
  │  │  • Used for: dashboards, reports, executive summaries  │  │
  │  │  Example: daily_revenue, customer_ltv, product_perf    │  │
  │  └─────────────────────┬─────────────────────────────────┘  │
  │                        │                                    │
  │  ┌─────────────────────▼─────────────────────────────────┐  │
  │  │  📊 CONSUMPTION                                        │  │
  │  │  Tableau │ Power BI │ Looker │ ML Models │ APIs        │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

This is called the "MEDALLION ARCHITECTURE" (Bronze → Silver → Gold)
It's the industry standard for lakehouse data pipelines.
```

### Implementing the Medallion Architecture

```sql
-- ==========================================
-- BRONZE LAYER: Raw ingestion
-- ==========================================
CREATE TABLE bronze_orders
USING DELTA
LOCATION 's3://lakehouse/bronze/orders/'
AS SELECT 
    *,
    current_timestamp() AS ingestion_time,
    input_file_name() AS source_file
FROM read_files('s3://raw-data/orders/', format => 'json');

-- ==========================================
-- SILVER LAYER: Cleaned + validated
-- ==========================================
CREATE TABLE silver_orders
USING DELTA
LOCATION 's3://lakehouse/silver/orders/'
AS SELECT
    CAST(order_id AS BIGINT) AS order_id,
    CAST(order_date AS DATE) AS order_date,
    CAST(customer_id AS INT) AS customer_id,
    CAST(amount AS DECIMAL(12,2)) AS amount,
    UPPER(TRIM(status)) AS status,
    ingestion_time
FROM bronze_orders
WHERE order_id IS NOT NULL          -- Remove nulls
  AND amount > 0                    -- Remove invalid amounts
  AND order_date >= '2020-01-01';   -- Remove ancient data

-- Deduplicate
CREATE OR REPLACE TABLE silver_orders AS
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY order_id ORDER BY ingestion_time DESC
    ) AS rn
    FROM silver_orders
) WHERE rn = 1;

-- ==========================================
-- GOLD LAYER: Business aggregations
-- ==========================================
CREATE TABLE gold_daily_revenue
USING DELTA
LOCATION 's3://lakehouse/gold/daily_revenue/'
AS SELECT
    o.order_date,
    c.region,
    p.category,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(o.amount) AS total_revenue,
    AVG(o.amount) AS avg_order_value
FROM silver_orders o
JOIN silver_customers c ON o.customer_id = c.customer_id
JOIN silver_products p ON o.product_id = p.product_id
GROUP BY o.order_date, c.region, p.category;
```

---

## 🔍 Query Engines for Lakehouse

You're not limited to Spark. Multiple engines can query lakehouse tables:

```
┌──────────────────┬──────────────────────────────────────────┐
│ Engine           │ Best For                                 │
├──────────────────┼──────────────────────────────────────────┤
│ Apache Spark     │ Heavy ETL, ML, batch processing         │
│ Databricks SQL   │ BI dashboards, ad-hoc SQL analytics     │
│ Trino (Presto)   │ Interactive federated queries           │
│ Dremio           │ Self-service analytics, data-as-a-service│
│ Apache Flink     │ Real-time streaming analytics           │
│ Snowflake        │ Iceberg tables via external catalogs    │
│ BigQuery         │ Iceberg/Delta via BigLake               │
│ AWS Athena       │ Serverless S3 queries (Iceberg/Delta)   │
│ Redshift Spectrum│ S3 data via external tables             │
│ StarRocks/Doris  │ Real-time OLAP on lakehouse data        │
└──────────────────┴──────────────────────────────────────────┘

🔥 TREND: Engine-agnostic formats (especially Iceberg) let you 
   use ANY engine on the SAME data — no lock-in!
```

---

## 🆚 Lakehouse vs Warehouse vs Lake — When to Use What

```
┌────────────────────────────────────────────────────────────────────┐
│  DECISION FRAMEWORK                                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Choose DATA WAREHOUSE (Snowflake, BigQuery, Redshift) when:       │
│  → Pure BI/analytics workload                                      │
│  → Structured data only                                            │
│  → Want zero administration                                        │
│  → Team knows SQL, not Spark                                       │
│  → < 100 TB of data                                                │
│                                                                    │
│  Choose DATA LAKEHOUSE (Delta/Iceberg + Spark) when:               │
│  → Mixed workloads (BI + ML + data engineering)                    │
│  → Massive scale (> 100 TB, petabyte range)                        │
│  → Need open formats (avoid vendor lock-in)                        │
│  → Unstructured data (images, audio, video, logs)                  │
│  → Real-time + batch in one platform                               │
│  → Cost-sensitive (S3 storage is 10x cheaper than warehouse)       │
│                                                                    │
│  Choose DATA LAKE (raw S3/ADLS) when:                              │
│  → Pure storage/landing zone                                       │
│  → Data science exploration (notebooks on raw data)                │
│  → Archive/backup                                                  │
│  → ⚠️ NOT for production analytics (upgrade to lakehouse)          │
│                                                                    │
│  🔥 MODERN TREND: Many companies use BOTH                          │
│  → Lakehouse for data engineering + ML                             │
│  → Warehouse for BI dashboards (query from lakehouse via Iceberg)  │
│  → "Lakehouse as the source of truth, warehouse as serving layer"  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🧪 Quick Knowledge Check

```
Q1: What problem does a lakehouse solve that data lakes couldn't?
A1: ACID transactions, schema enforcement, UPDATE/DELETE/MERGE,
    time travel, and data quality — all missing in raw data lakes.

Q2: What's the difference between Delta Lake and Apache Iceberg?
A2: Both add ACID + time travel to data lakes. Key difference: Iceberg 
    has hidden partitioning and partition evolution (change partitions 
    without rewriting data). Delta is more mature in Spark/Databricks.

Q3: What is the Medallion Architecture?
A3: Bronze (raw) → Silver (cleaned) → Gold (aggregated/business-ready).
    Standard pattern for organizing lakehouse data pipelines.

Q4: Why is Spark faster than Hadoop MapReduce?
A4: In-memory processing. MapReduce writes to disk between steps,
    Spark keeps intermediate results in RAM → 100x faster.

Q5: Can Snowflake/BigQuery read lakehouse tables?
A5: Yes! Snowflake reads Iceberg tables via external catalogs.
    BigQuery reads Delta/Iceberg via BigLake. Engines are converging.

Q6: Copy-on-Write vs Merge-on-Read in Hudi?
A6: CoW: rewrites file on update (slow write, fast read).
    MoR: writes delta log (fast write, merge on read).
```

---

## 🗺️ Chapter Summary

```
┌────────────────────────────────────────────────────────┐
│  SPARK SQL & DATA LAKEHOUSE — KEY TAKEAWAYS            │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ✅ Evolution: Warehouse → Lake → Lakehouse            │
│  ✅ Spark: distributed engine, 100x faster than MR     │
│  ✅ Spark SQL: standard SQL on petabyte-scale data     │
│  ✅ Lakehouse = cheap storage + warehouse reliability  │
│  ✅ Delta Lake: ACID on data lakes (Databricks)        │
│  ✅ Apache Iceberg: engine-agnostic, hidden partitions │
│  ✅ Apache Hudi: real-time upserts (Uber)              │
│  ✅ Medallion: Bronze → Silver → Gold pipeline         │
│  ✅ ACID, time travel, MERGE on data lakes             │
│  ✅ Open formats: no vendor lock-in                    │
│                                                        │
│  🔥 INTERVIEW ESSENTIALS:                              │
│     Lake vs Warehouse vs Lakehouse, Delta vs Iceberg,  │
│     Medallion Architecture, Spark vs MapReduce,        │
│     ACID on data lakes, partition evolution             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 🗺️ Part 6 Complete — What's Next?

```
You've now mastered:
✅ 6.1 — Data Warehouse Concepts (OLTP vs OLAP, Star Schema, SCD)
✅ 6.2 — Amazon Redshift (MPP, distribution, sort keys)
✅ 6.3 — Google BigQuery (serverless, partitioning, BQML)
✅ 6.4 — Snowflake (virtual warehouses, cloning, data sharing)
✅ 6.5 — Apache Spark SQL & Data Lakehouse (Delta, Iceberg, Hudi)

→ Next: Part 7 — Database Architecture & System Design
```

---

*"The lakehouse is not just a technology — it's the realization that we never needed separate systems for storing and analyzing data."*
