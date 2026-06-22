# 6.3 — Google BigQuery 🟡🔥

> **"BigQuery is the database equivalent of ordering food delivery — you don't own the kitchen, you don't hire the chef, but you get a gourmet meal in seconds."**

> **Level:** 🟡 Intermediate | 🔥 High Demand  
> **Time to Master:** ~4-5 hours  
> **Prerequisites:** Chapter 6.1 (Data Warehouse Concepts)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand BigQuery's **serverless architecture** — Dremel, Colossus, Jupiter, Borg
- Master **partitioning** and **clustering** to slash query costs by 90%
- Write queries with **Standard SQL** + BigQuery-specific features
- Use **streaming inserts** for real-time data ingestion
- Build **ML models** directly inside BigQuery (no Python needed!)
- Optimize for **cost** (pay per TB scanned) and **performance**

---

## 🧠 What is Google BigQuery?

```
BigQuery = Fully serverless, petabyte-scale analytics data warehouse by Google

Key Facts:
┌──────────────────────────────────────────────────────────┐
│ • 100% SERVERLESS — no nodes, no clusters, no infra     │
│ • Pay per QUERY (per TB of data scanned) + storage      │
│ • Processes petabytes in SECONDS                         │
│ • No indexes, no VACUUM, no tuning knobs                │
│ • Standard SQL (ANSI 2011 compliant)                     │
│ • Born from Google's internal Dremel system (2006)       │
│ • Public launch: 2011                                    │
│ • Used by: Spotify, Twitter/X, HSBC, Snap, Wayfair      │
│ • Free tier: 1 TB querying + 10 GB storage/month         │
└──────────────────────────────────────────────────────────┘

Why "BigQuery"?
→ It literally handles BIG queries — scanning terabytes in seconds
→ Google needed this internally to analyze web-scale data
→ They productized it for everyone
```

---

## 🏗️ Architecture — The 4 Pillars of BigQuery

BigQuery's architecture is unlike ANY traditional database. It separates everything.

```
┌─────────────────────────────────────────────────────────────────┐
│                    BIGQUERY ARCHITECTURE                         │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  🧮 DREMEL — The Execution Engine                         │  │
│  │  • Breaks queries into tiny tasks                         │  │
│  │  • Distributes to thousands of workers (slots)            │  │
│  │  • Tree architecture: root → mixers → leaf workers        │  │
│  │  • Each leaf reads a small chunk of data                  │  │
│  │  • Results bubble up through the tree                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│                           │ Reads data from                     │
│                           ▼                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  📦 COLOSSUS — The Storage System                         │  │
│  │  • Google's distributed file system (successor to GFS)    │  │
│  │  • Stores data in columnar format (Capacitor)             │  │
│  │  • Automatic replication (durability)                     │  │
│  │  • Automatic compression & encryption                     │  │
│  │  • Storage is COMPLETELY separate from compute            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│                           │ Connected via                       │
│                           ▼                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  🌐 JUPITER — The Network                                │  │
│  │  • Google's petabit-scale network                         │  │
│  │  • 1 Petabit/sec bisection bandwidth                      │  │
│  │  • This is why compute & storage can be separated!        │  │
│  │  • Fast enough that remote storage feels local            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│                           │ Orchestrated by                     │
│                           ▼                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  🤖 BORG — The Orchestrator                               │  │
│  │  • Google's cluster management system                     │  │
│  │  • Allocates compute resources (slots) dynamically        │  │
│  │  • Predecessor to Kubernetes                              │  │
│  │  • Ensures fault tolerance and resource efficiency        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How a Query Actually Executes

```
You run: SELECT category, SUM(revenue) FROM sales GROUP BY category;

Step 1: ROOT SERVER receives query, parses SQL, creates execution plan

Step 2: MIXER NODES distribute work to LEAF WORKERS
                    
                    ┌──────────────┐
                    │  ROOT SERVER  │  ← Aggregates final result
                    └──────┬───────┘
                  ┌────────┼────────┐
             ┌────▼───┐┌───▼────┐┌──▼─────┐
             │Mixer 1 ││Mixer 2 ││Mixer 3 │  ← Partial aggregations
             └──┬──┬──┘└──┬──┬──┘└──┬──┬──┘
             ┌──┘  └──┐┌──┘  └──┐┌──┘  └──┐
           ┌─▼─┐  ┌──▼┐▼─┐  ┌─▼┐▼──┐ ┌──▼┐
           │L1 │  │L2 ││L3│  │L4││L5│ │L6 │  ← Read from Colossus
           └───┘  └───┘└──┘  └──┘└──┘ └───┘    (thousands of workers)

Step 3: Each LEAF reads its portion of data from Colossus
        → Only reads the "category" and "revenue" COLUMNS (columnar!)
        → Computes local SUM per category

Step 4: MIXERS combine partial results from their leaf workers

Step 5: ROOT combines all mixer results → Final answer returned

Total time: 3-10 seconds for TB of data
Workers used: Could be 1,000 to 10,000+ (all managed by Google)
```

> 💡 **Key Insight**: You NEVER see or manage these workers. You just write SQL and pay per TB scanned. Google handles all the infrastructure.

---

## 💰 Pricing Model — Pay Per Query

This is the **most unique** aspect of BigQuery. You pay for what you scan, not what you provision.

```
┌──────────────────────────────────────────────────────────────┐
│  BIGQUERY PRICING                                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  📊 ON-DEMAND PRICING (Default)                              │
│  ───────────────────────────────                             │
│  • $6.25 per TB of data scanned                              │
│  • First 1 TB/month is FREE                                  │
│  • If your query scans 100 GB → pay $0.625                   │
│  • If your query scans 5 TB → pay $31.25                     │
│                                                              │
│  📦 STORAGE PRICING                                          │
│  ───────────────────────────                                 │
│  • Active storage: $0.02/GB/month                            │
│  • Long-term (>90 days untouched): $0.01/GB/month            │
│  • First 10 GB/month is FREE                                 │
│                                                              │
│  🏢 FLAT-RATE / EDITIONS (For predictable workloads)         │
│  ───────────────────────────────────────────                  │
│  • Buy "slots" (compute units) — fixed monthly price         │
│  • Standard/Enterprise/Enterprise Plus editions              │
│  • $0.04/slot-hour (Standard Edition, auto-scaling)          │
│  • Better for teams running many queries daily               │
│                                                              │
└──────────────────────────────────────────────────────────────┘

🔥 COST EXAMPLE:
Your table: fact_sales (500 GB, 10 columns)

Bad query:   SELECT * FROM fact_sales;
→ Scans ALL 500 GB → Cost: $3.12

Good query:  SELECT date, SUM(amount) FROM fact_sales GROUP BY date;
→ Scans ONLY date + amount columns (~100 GB) → Cost: $0.62

Better query (partitioned table):
→ SELECT date, SUM(amount) FROM fact_sales 
  WHERE date BETWEEN '2026-01-01' AND '2026-03-31' GROUP BY date;
→ Scans only Q1 partition (~25 GB) → Cost: $0.15

💡 PARTITION + SELECT SPECIFIC COLUMNS = 95% cost reduction!
```

---

## 📐 Partitioning — Slash Costs & Speed Up Queries

Partitioning divides a table into segments so BigQuery only scans relevant data.

```
WITHOUT PARTITIONING:
┌──────────────────────────────────────────────┐
│  fact_sales (500 GB — ALL in one chunk)       │
│  2023 data + 2024 data + 2025 data + 2026    │
└──────────────────────────────────────────────┘
Query: WHERE date >= '2026-01-01'
→ Scans ALL 500 GB (even though 2026 = only 50 GB) 🐌💸

WITH PARTITIONING BY DATE:
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  2023 data │ │  2024 data │ │  2025 data │ │  2026 data │
│  (120 GB)  │ │  (130 GB)  │ │  (150 GB)  │ │  (100 GB)  │
└────────────┘ └────────────┘ └────────────┘ └────────────┘
     SKIP ✅       SKIP ✅       SKIP ✅      SCAN ONLY THIS 🔍
     
Query: WHERE date >= '2026-01-01'
→ Scans ONLY 100 GB (2026 partition) → 5x faster, 5x cheaper! ⚡💰
```

### Partition Types

```sql
-- 1. TIME-UNIT PARTITIONING (Most common)
CREATE TABLE fact_sales
PARTITION BY DATE(order_date)     -- DAY granularity (default)
AS SELECT * FROM raw_sales;

-- Or with explicit granularity:
CREATE TABLE fact_sales (
    order_id    INT64,
    order_date  DATE,
    amount      FLOAT64
)
PARTITION BY DATE_TRUNC(order_date, MONTH);  -- MONTH granularity

-- 2. INTEGER RANGE PARTITIONING
CREATE TABLE fact_events (
    user_id     INT64,
    event_type  STRING,
    event_data  STRING
)
PARTITION BY RANGE_BUCKET(user_id, GENERATE_ARRAY(0, 1000000, 10000));
-- Creates partitions: 0-9999, 10000-19999, 20000-29999, ...

-- 3. INGESTION TIME PARTITIONING
CREATE TABLE streaming_events (
    event_id    STRING,
    payload     STRING
)
PARTITION BY _PARTITIONDATE;  -- Partitioned by when data was loaded
```

### Partition Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│  PARTITIONING RULES                                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Always partition large tables (>1 GB)                    │
│  ✅ Use DATE/TIMESTAMP columns (most queries filter by time) │
│  ✅ Require partition filter in queries:                     │
│     ALTER TABLE fact_sales                                   │
│     SET OPTIONS (require_partition_filter = true);           │
│     → Prevents accidental full-table scans!                  │
│                                                              │
│  ❌ Don't create too many partitions (limit: 4,000/table)    │
│  ❌ Don't partition by high-cardinality (user_id → millions  │
│     of tiny partitions)                                      │
│  ❌ Don't use DAY if MONTH is sufficient                     │
│     → 365 partitions/year vs 12 partitions/year              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 Clustering — Sort Data Within Partitions

Clustering physically **reorders** data within each partition based on specified columns.

```
PARTITIONING answers: "WHICH partition to scan?"
CLUSTERING  answers: "WITHIN that partition, which blocks to scan?"

PARTITIONED + CLUSTERED:
┌──────────────────── Partition: January 2026 ──────────────────────┐
│                                                                    │
│  Cluster by: category, region                                      │
│                                                                    │
│  Block 1: category="Electronics", region="Asia"     (10 GB)       │
│  Block 2: category="Electronics", region="Europe"   (8 GB)        │
│  Block 3: category="Fashion", region="Asia"         (12 GB)       │
│  Block 4: category="Fashion", region="Europe"       (6 GB)        │
│  Block 5: category="Food", region="Americas"        (9 GB)        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

Query: WHERE date = '2026-01-15' AND category = 'Electronics'
→ Partition pruning: only scan January 2026
→ Cluster pruning: only scan Blocks 1 & 2 (skip 3,4,5)
→ Result: scan 18 GB instead of 45 GB → 60% less data scanned!
```

```sql
-- Create partitioned AND clustered table
CREATE TABLE fact_sales (
    sale_id       INT64,
    order_date    DATE,
    customer_id   INT64,
    product_id    INT64,
    category      STRING,
    region        STRING,
    amount        FLOAT64,
    quantity      INT64
)
PARTITION BY DATE_TRUNC(order_date, MONTH)
CLUSTER BY category, region;

-- Up to 4 clustering columns allowed
-- Order matters: first column = most filtering benefit
-- BigQuery auto-reclusters data (no manual maintenance!)
```

### Partitioning vs Clustering Decision

```
┌──────────────────────────────────────────────────────────────┐
│  WHEN TO USE WHAT                                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Use PARTITIONING when:                                      │
│  → Filter/GROUP BY on DATE columns (almost always)           │
│  → You want strict cost control (partition pruning)          │
│  → Filter column has < 4,000 distinct values                 │
│                                                              │
│  Use CLUSTERING when:                                        │
│  → Filter columns have HIGH cardinality                      │
│  → Multiple columns used in WHERE/JOIN                       │
│  → Partition alone isn't enough                              │
│                                                              │
│  Use BOTH (most common pattern):                             │
│  → PARTITION BY date + CLUSTER BY category, region           │
│  → Best cost & performance combination                       │
│                                                              │
│  Use NEITHER when:                                           │
│  → Table is tiny (< 1 GB)                                   │
│  → Queries always scan everything                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 📥 Loading Data into BigQuery

### Batch Loading (Free!)

```sql
-- Load from Google Cloud Storage (GCS) — MOST COMMON
LOAD DATA INTO fact_sales
FROM FILES (
  format = 'PARQUET',
  uris = ['gs://my-bucket/sales/2026/*.parquet']
);

-- Load from CSV
bq load \
  --source_format=CSV \
  --skip_leading_rows=1 \
  my_dataset.fact_sales \
  gs://my-bucket/sales/data.csv \
  sale_id:INTEGER,order_date:DATE,amount:FLOAT

-- Supported formats: CSV, JSON, Avro, Parquet, ORC, 
--                    Datastore exports, Firestore exports

-- 💡 PARQUET is recommended:
-- ✅ Columnar (matches BigQuery storage)
-- ✅ Compressed (less data to transfer)
-- ✅ Schema embedded (no separate schema definition)
-- ✅ Type-safe (no CSV parsing errors)
```

### Streaming Inserts (Real-Time)

```sql
-- For real-time data (IoT, clickstream, logs)
-- Use the BigQuery Storage Write API or legacy streaming API

-- Python example (using google-cloud-bigquery library):
-- rows = [
--     {"sale_id": 1001, "order_date": "2026-06-02", "amount": 999.00},
--     {"sale_id": 1002, "order_date": "2026-06-02", "amount": 499.00},
-- ]
-- table_id = "project.dataset.fact_sales"
-- errors = client.insert_rows_json(table_id, rows)

-- Streaming pricing: $0.01 per 200 MB (for legacy streaming API)
-- Storage Write API: FREE for committed writes!

-- 💡 For most use cases, batch loading at intervals (hourly/daily) 
--    is cheaper than streaming
```

### Query External Data (Federated Queries)

```sql
-- Query data in GCS without loading it (like Redshift Spectrum)
CREATE EXTERNAL TABLE my_dataset.external_logs
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-bucket/logs/2026/*.parquet']
);

SELECT * FROM my_dataset.external_logs WHERE date = '2026-06-02';

-- ⚠️ External tables are SLOWER than native BigQuery tables
-- Use for infrequent queries on cold data
-- For hot data, load into native BigQuery tables
```

---

## 🧪 BigQuery SQL — Unique Features

BigQuery uses **Standard SQL** with some powerful extensions:

### STRUCT and ARRAY — Nested Data

```sql
-- BigQuery supports NESTED and REPEATED fields
-- No need to flatten JSON — store it naturally!

CREATE TABLE orders (
    order_id    INT64,
    order_date  DATE,
    customer    STRUCT<
        id      INT64,
        name    STRING,
        email   STRING
    >,
    items       ARRAY<STRUCT<
        product_id   INT64,
        product_name STRING,
        quantity     INT64,
        price        FLOAT64
    >>
);

-- Query nested data
SELECT 
    order_id,
    customer.name AS customer_name,
    item.product_name,
    item.quantity * item.price AS line_total
FROM orders,
UNNEST(items) AS item
WHERE order_date = '2026-06-02';

-- Why this matters:
-- ✅ No JOINs needed (data is pre-joined in the structure)
-- ✅ Reads less data (no foreign key lookups)
-- ✅ Perfect for event data, logs, analytics
```

### APPROX Functions — Speed Over Precision

```sql
-- Exact count: scans ALL data → expensive
SELECT COUNT(DISTINCT user_id) FROM events;  -- 30 seconds, costs $X

-- Approximate count: uses HyperLogLog → fast + cheap
SELECT APPROX_COUNT_DISTINCT(user_id) FROM events;  -- 3 seconds, same cost
-- Accuracy: within 1-2% (good enough for dashboards!)

-- Other APPROX functions:
SELECT APPROX_QUANTILES(amount, 100) FROM sales;      -- Percentiles
SELECT APPROX_TOP_COUNT(category, 10) FROM sales;     -- Top N values
SELECT APPROX_TOP_SUM(product, revenue, 10) FROM sales; -- Top N by sum
```

### Wildcard Tables — Query Multiple Tables at Once

```sql
-- Query all monthly tables at once
SELECT * FROM `project.dataset.events_*`
WHERE _TABLE_SUFFIX BETWEEN '202601' AND '202612';

-- Expands to: events_202601, events_202602, ... events_202612
-- 💡 Useful for legacy table-per-date designs
```

### Time Travel — Query Past Data

```sql
-- Query the table as it was 1 hour ago
SELECT * FROM fact_sales
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR);

-- Query table as it was at a specific time
SELECT * FROM fact_sales
FOR SYSTEM_TIME AS OF '2026-06-01 10:00:00 UTC';

-- Time travel window: 7 days (default, can extend to 7 days max)
-- 💡 Great for: "Oops, I deleted rows!" → recover from time travel
```

### Scripting & Procedures

```sql
-- BigQuery supports multi-statement scripts
DECLARE target_date DATE DEFAULT CURRENT_DATE();
DECLARE threshold FLOAT64;

SET threshold = (SELECT AVG(amount) * 2 FROM fact_sales);

-- Use in query
SELECT * FROM fact_sales 
WHERE order_date = target_date AND amount > threshold;

-- Stored procedures
CREATE OR REPLACE PROCEDURE my_dataset.refresh_summary(IN p_date DATE)
BEGIN
    DELETE FROM daily_summary WHERE summary_date = p_date;
    
    INSERT INTO daily_summary
    SELECT p_date, category, SUM(amount), COUNT(*)
    FROM fact_sales
    WHERE order_date = p_date
    GROUP BY category;
END;

CALL my_dataset.refresh_summary('2026-06-02');
```

---

## 🤖 BigQuery ML — Machine Learning in SQL!

This is BigQuery's **killer feature**. Build ML models using SQL — no Python, no TensorFlow, no data export.

```sql
-- Step 1: CREATE a model (train it)
CREATE OR REPLACE MODEL my_dataset.predict_revenue
OPTIONS (
    model_type = 'LINEAR_REG',       -- Linear Regression
    input_label_cols = ['revenue']    -- What we're predicting
) AS
SELECT 
    category,
    region,
    month,
    day_of_week,
    is_holiday,
    promo_discount,
    revenue                          -- Label (target variable)
FROM training_data
WHERE year BETWEEN 2023 AND 2025;

-- Step 2: EVALUATE the model
SELECT * FROM ML.EVALUATE(MODEL my_dataset.predict_revenue);
-- Returns: mean_absolute_error, mean_squared_error, r2_score, etc.

-- Step 3: PREDICT on new data
SELECT *
FROM ML.PREDICT(MODEL my_dataset.predict_revenue, (
    SELECT category, region, month, day_of_week, is_holiday, promo_discount
    FROM fact_sales
    WHERE year = 2026 AND month = 7
));

-- Supported model types:
-- ✅ LINEAR_REG          → Predict numbers (revenue, price)
-- ✅ LOGISTIC_REG        → Classify (yes/no, churn prediction)
-- ✅ KMEANS              → Clustering (customer segments)
-- ✅ BOOSTED_TREE_CLASSIFIER/REGRESSOR → XGBoost
-- ✅ DNN_CLASSIFIER/REGRESSOR → Deep Neural Networks  
-- ✅ AUTOML_CLASSIFIER/REGRESSOR → Google AutoML
-- ✅ ARIMA_PLUS          → Time series forecasting
-- ✅ MATRIX_FACTORIZATION → Recommendation systems
-- ✅ TENSORFLOW          → Import TF SavedModel
-- ✅ TRANSFORM           → Feature engineering in SQL

-- 🔥 This means a SQL analyst can build ML models
--    without knowing Python or ML frameworks!
```

---

## 🛡️ Security & Access Control

```
┌──────────────────────────────────────────────────────────┐
│  BIGQUERY SECURITY MODEL                                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  🏢 Organization Level                                   │
│   └── 📁 Project Level                                   │
│        └── 📦 Dataset Level ← Primary access boundary    │
│             └── 📋 Table Level                            │
│                  └── 🔒 Column Level (column policies)    │
│                       └── 🔐 Row Level (row policies)    │
│                                                          │
│  IAM Roles:                                              │
│  • BigQuery Admin → Full control                         │
│  • BigQuery Data Editor → Read + Write data              │
│  • BigQuery Data Viewer → Read only                      │
│  • BigQuery Job User → Run queries (no data access)      │
│  • BigQuery User → Run queries + list datasets           │
│                                                          │
│  Column-Level Security:                                   │
│  → Tag sensitive columns (SSN, salary)                   │
│  → Only authorized users see those columns               │
│  → Others see NULL for restricted columns                │
│                                                          │
│  Row-Level Security:                                      │
│  → Filter rows based on user attributes                  │
│  → "Sales reps only see their region's data"             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

```sql
-- Column-level security using policy tags
-- Step 1: Create a taxonomy and policy tag in Data Catalog (UI/API)
-- Step 2: Assign tag to sensitive columns
-- Step 3: Grant access to specific users

-- Row-level security
CREATE ROW ACCESS POLICY region_filter
ON my_dataset.fact_sales
GRANT TO ('user:analyst@company.com', 'group:sales-east@company.com')
FILTER USING (region = 'East');

-- analyst@company.com can ONLY see rows where region = 'East'
```

---

## ⚡ Query Optimization — Reduce Cost & Increase Speed

### The Golden Rules

```
┌──────────────────────────────────────────────────────────────┐
│  BIGQUERY OPTIMIZATION RULES                                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. SELECT ONLY NEEDED COLUMNS                               │
│     ❌ SELECT * FROM fact_sales               (scans 500 GB) │
│     ✅ SELECT date, SUM(amount) FROM fact_sales (scans 50 GB)│
│     → Columnar storage = only read requested columns         │
│                                                              │
│  2. USE PARTITION FILTERS                                    │
│     ❌ SELECT ... FROM fact_sales              (all years)    │
│     ✅ SELECT ... WHERE order_date >= '2026-01-01'           │
│     → Prunes entire partitions                               │
│                                                              │
│  3. AVOID SELECT DISTINCT ON LARGE TABLES                    │
│     ❌ SELECT DISTINCT user_id FROM events     (scans all)   │
│     ✅ Use APPROX_COUNT_DISTINCT for counts                  │
│     ✅ Use GROUP BY instead of DISTINCT                       │
│                                                              │
│  4. USE CLUSTERING FOR NON-DATE FILTERS                      │
│     → CLUSTER BY category, region                            │
│     → Skips irrelevant data blocks                           │
│                                                              │
│  5. AVOID CROSS JOINS                                        │
│     → Cartesian products explode data → expensive            │
│                                                              │
│  6. MATERIALIZE INTERMEDIATE RESULTS                         │
│     → Use CREATE TABLE AS SELECT for reused subqueries       │
│     → CTE is re-executed each time (not cached)              │
│                                                              │
│  7. PREVIEW QUERY COST BEFORE RUNNING                        │
│     → BigQuery UI shows estimated bytes scanned              │
│     → Use --dry_run flag in bq CLI                           │
│     bq query --dry_run 'SELECT ...'                          │
│                                                              │
│  8. SET MAXIMUM BYTES BILLED                                 │
│     → Fail query if it exceeds budget:                       │
│     SET @@max_bytes_billed = 1000000000;  -- 1 GB max        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Query Execution Plan — INFORMATION_SCHEMA

```sql
-- Check query execution details
SELECT
    job_id,
    query,
    total_bytes_processed / POW(1024,3) AS gb_scanned,
    total_slot_ms / 1000 AS slot_seconds,
    cache_hit,
    creation_time,
    total_bytes_billed / POW(1024,4) AS tb_billed,
    total_bytes_billed / POW(1024,4) * 6.25 AS estimated_cost_usd
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
ORDER BY total_bytes_processed DESC
LIMIT 20;

-- 💡 Cache hit = TRUE means BigQuery used cached results (FREE!)
-- Cache expires after 24 hours or when underlying table changes
```

---

## 🔄 Scheduling & Automation

```sql
-- Scheduled queries (run SQL on a schedule)
-- Created via BigQuery UI or API

-- Example: Daily summary refresh
-- Schedule: Every day at 2:00 AM UTC
-- Query:
MERGE daily_summary AS target
USING (
    SELECT 
        CURRENT_DATE() AS summary_date,
        category,
        SUM(amount) AS total_revenue,
        COUNT(*) AS order_count
    FROM fact_sales
    WHERE order_date = CURRENT_DATE() - 1
    GROUP BY category
) AS source
ON target.summary_date = source.summary_date 
   AND target.category = source.category
WHEN MATCHED THEN UPDATE SET 
    total_revenue = source.total_revenue,
    order_count = source.order_count
WHEN NOT MATCHED THEN INSERT VALUES (
    source.summary_date, source.category, 
    source.total_revenue, source.order_count
);
```

---

## 🆚 BigQuery vs Others — When to Choose BigQuery

```
Choose BigQuery when:
✅ You're on Google Cloud (GCP)
✅ You want ZERO infrastructure management
✅ Workloads are sporadic/unpredictable (pay per query)
✅ You need built-in ML (BQML)
✅ You work with nested/repeated data (STRUCT/ARRAY)
✅ You want geospatial analytics (BigQuery GIS)
✅ You need real-time streaming inserts

Choose something else when:
❌ You're heavily invested in AWS (→ Redshift)
❌ You need multi-cloud (→ Snowflake)
❌ You need sub-second query latency (→ Druid, ClickHouse)
❌ You have constant, predictable high workloads (flat-rate may be cheaper)
❌ You need OLTP transactions (→ Cloud SQL, Spanner)
```

---

## 🧪 Quick Knowledge Check

```
Q1: How does BigQuery charge for queries?
A1: On-demand: $6.25 per TB of data SCANNED. Partition + cluster + select 
    specific columns to minimize scanning.

Q2: What's the difference between partitioning and clustering?
A2: Partitioning: separates table into large chunks (by date/range).
    Clustering: sorts data WITHIN partitions (by category, region).
    Use both for maximum performance.

Q3: Can BigQuery train ML models?
A3: Yes! BigQuery ML (BQML) trains models using SQL.
    CREATE MODEL + ML.PREDICT. Supports linear regression, XGBoost,
    deep learning, time series, and more.

Q4: What are slots?
A4: Slots = BigQuery compute units (virtual CPUs). On-demand gets 2,000 
    slots by default. Flat-rate lets you buy dedicated slots.

Q5: How does BigQuery handle nested data?
A5: STRUCT for nested fields, ARRAY for repeated fields.
    UNNEST() to flatten arrays in queries. No JOINs needed for 
    pre-joined nested data.

Q6: How long does time travel work in BigQuery?
A6: Up to 7 days. Query historical data using FOR SYSTEM_TIME AS OF.
```

---

## 🗺️ Chapter Summary

```
┌────────────────────────────────────────────────────────┐
│  GOOGLE BIGQUERY — KEY TAKEAWAYS                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ✅ 100% serverless — no nodes, no clusters            │
│  ✅ Dremel (compute) + Colossus (storage) separated    │
│  ✅ Pay per TB scanned (or buy slots)                  │
│  ✅ Partition by date → prune entire time ranges       │
│  ✅ Cluster by category/region → skip blocks           │
│  ✅ STRUCT + ARRAY → nested data without JOINs         │
│  ✅ BigQuery ML → train models in SQL                  │
│  ✅ Time travel → query past data (up to 7 days)       │
│  ✅ Streaming inserts for real-time data               │
│  ✅ Auto-manages compression, optimization, scaling    │
│                                                        │
│  🔥 INTERVIEW ESSENTIALS:                              │
│     Serverless architecture, pricing model,            │
│     partitioning vs clustering, BQML,                  │
│     cost optimization techniques                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [6.4 — Snowflake — The Cloud Data Platform →](./04-Snowflake.md)

---

*"BigQuery made data warehousing as easy as writing SQL — Google handles the rest."*
