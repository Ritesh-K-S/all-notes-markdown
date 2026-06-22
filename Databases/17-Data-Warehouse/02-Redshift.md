# 6.2 — Amazon Redshift 🟡🔥

> **"Redshift is what happens when Amazon takes PostgreSQL, turns it sideways (columnar), and throws a petabyte of data at it."**

> **Level:** 🟡 Intermediate | 🔥 High Demand  
> **Time to Master:** ~4-5 hours  
> **Prerequisites:** Chapter 6.1 (Data Warehouse Concepts)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand Redshift's **architecture** — nodes, slices, and how it differs from PostgreSQL
- Choose the right **distribution style** (KEY, ALL, EVEN, AUTO) for your tables
- Design **sort keys** that make queries fly
- Use **Redshift Spectrum** to query data directly in S3
- Optimize queries with **compression encodings**, **WLM**, and **concurrency scaling**
- Know when to use Redshift vs BigQuery vs Snowflake

---

## 🧠 What is Amazon Redshift?

```
Amazon Redshift = Fully managed, petabyte-scale data warehouse in AWS

Key Facts:
┌──────────────────────────────────────────────────────┐
│ • Based on PartiQL/PostgreSQL 8.x (but VERY modified)│
│ • Column-oriented storage (not row-based like PG)    │
│ • Massively Parallel Processing (MPP) architecture   │
│ • Can handle petabytes of data                       │
│ • ~$0.25/hour for a single node (starting price)     │
│ • Used by: Netflix, Airbnb, Lyft, Nasdaq, McDonald's │
│ • Launched: 2012 — Changed the DW market forever     │
└──────────────────────────────────────────────────────┘

Why the name "Redshift"?
→ In astronomy, redshift = shift toward red spectrum (moving away)
→ Amazon wanted you to "shift away" from traditional (expensive) warehouses
→ Oracle Exadata: ~$1M+/year → Redshift: ~$1K/year for same data 💰
```

---

## 🏗️ Architecture — How Redshift Works Internally

```
                        ┌────────────────────┐
                        │   CLIENT / BI Tool │
                        │  (Tableau, Python,  │
                        │   SQL Workbench)    │
                        └────────┬───────────┘
                                 │ JDBC/ODBC
                                 ▼
                   ┌──────────────────────────┐
                   │      LEADER NODE          │
                   │  • Receives SQL queries   │
                   │  • Parses & optimizes     │
                   │  • Creates execution plan │
                   │  • Distributes to compute │
                   │  • Aggregates results     │
                   │  • Returns to client      │
                   │  • Does NOT store data    │
                   └──────────┬───────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ COMPUTE    │  │ COMPUTE    │  │ COMPUTE    │
     │ NODE 1     │  │ NODE 2     │  │ NODE 3     │
     │            │  │            │  │            │
     │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────────┐ │
     │ │Slice 1 │ │  │ │Slice 1 │ │  │ │Slice 1 │ │
     │ │ (vCPU) │ │  │ │ (vCPU) │ │  │ │ (vCPU) │ │
     │ ├────────┤ │  │ ├────────┤ │  │ ├────────┤ │
     │ │Slice 2 │ │  │ │Slice 2 │ │  │ │Slice 2 │ │
     │ │ (vCPU) │ │  │ │ (vCPU) │ │  │ │ (vCPU) │ │
     │ └────────┘ │  │ └────────┘ │  │ └────────┘ │
     │   Local    │  │   Local    │  │   Local    │
     │   Storage  │  │   Storage  │  │   Storage  │
     └────────────┘  └────────────┘  └────────────┘

Each SLICE:
• Has its own portion of the data
• Processes queries in parallel
• Has dedicated memory and disk
• Executes the leader node's plan independently
```

### Node Types

```
┌──────────────────────────────────────────────────────────────┐
│  NODE TYPE COMPARISON                                        │
├───────────────┬────────────────┬─────────────────────────────┤
│               │ RA3 (Latest)   │ DC2 (Dense Compute)         │
├───────────────┼────────────────┼─────────────────────────────┤
│ Storage       │ Managed (S3)   │ Local SSD                   │
│ Scale compute │ Independent    │ Tied to storage             │
│   & storage   │ of storage     │                             │
│ Best for      │ Most workloads │ <1TB hot data               │
│ Caching       │ Local SSD cache│ N/A (all local)             │
│ Price model   │ Per node +     │ Per node (all-inclusive)     │
│               │ per storage    │                             │
│ Recommendation│ ✅ DEFAULT     │ Small, performance-critical  │
│               │   CHOICE       │ workloads                   │
└───────────────┴────────────────┴─────────────────────────────┘

RA3 Key Insight:
→ Data lives in S3 (unlimited, cheap)
→ Frequently accessed data cached on local SSD
→ Scale compute without scaling storage (and vice versa)
→ This is why RA3 is the modern default
```

### Redshift Serverless — The Simpler Option

```
Traditional Redshift            Redshift Serverless
(Provisioned)                   (Auto-managed)
┌──────────────────┐            ┌──────────────────┐
│ You choose:      │            │ AWS handles:     │
│ • Node type      │            │ • Node type      │
│ • Number of nodes│            │ • Scaling         │
│ • Cluster config │            │ • Capacity        │
│                  │            │                  │
│ ✅ Full control  │            │ ✅ Zero admin     │
│ ✅ Predictable   │            │ ✅ Auto-scales    │
│    pricing       │            │ ✅ Pay per query  │
│ ❌ Capacity      │            │ ❌ Less control   │
│    planning      │            │ ❌ Can be costlier│
│                  │            │    at scale       │
└──────────────────┘            └──────────────────┘

Start with Serverless for:
→ POCs, small teams, variable workloads, getting started fast

Switch to Provisioned when:
→ Predictable workloads, cost optimization needed, >10TB data
```

---

## 📐 Distribution Styles — WHERE Data Lives

This is the **#1 performance factor** in Redshift. Choosing the wrong distribution style can make queries **100x slower**.

### Why Distribution Matters

```
When Redshift executes a JOIN, it needs matching rows on the SAME node.
If matching rows are on DIFFERENT nodes → data must be SHUFFLED across the network.
Network = SLOW. Avoid shuffles.

BAD Distribution (data shuffle needed):
  Node 1: Orders [1-100]     Node 2: Orders [101-200]
  Node 1: Customers [A-M]    Node 2: Customers [N-Z]
  
  JOIN orders ON customers → Customer "Alex" is on Node 1,
  but some of Alex's orders might be on Node 2 → SHUFFLE! 🐌

GOOD Distribution (no shuffle needed):
  Node 1: Orders for Customers [A-M] + Customers [A-M]
  Node 2: Orders for Customers [N-Z] + Customers [N-Z]
  
  JOIN → Everything needed is LOCAL → NO shuffle! ⚡
```

### The 4 Distribution Styles

```
1. KEY Distribution
   → Rows with the SAME key value go to the SAME node
   → Use for large tables that frequently JOIN on that key
   
   DISTSTYLE KEY DISTKEY(customer_id)
   
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   Node 1    │  │   Node 2    │  │   Node 3    │
   │ cust A,B,C  │  │ cust D,E,F  │  │ cust G,H,I  │
   │ + orders    │  │ + orders    │  │ + orders    │
   │ for A,B,C   │  │ for D,E,F   │  │ for G,H,I   │
   └─────────────┘  └─────────────┘  └─────────────┘
   
   ✅ Co-locates JOIN data → no network shuffle
   ❌ Skew risk: if one key has 80% of data → one node overloaded

2. ALL Distribution
   → ENTIRE table copied to EVERY node
   → Use for SMALL dimension tables (< 5M rows)
   
   DISTSTYLE ALL
   
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   Node 1    │  │   Node 2    │  │   Node 3    │
   │ Full copy   │  │ Full copy   │  │ Full copy   │
   │ of dim_date │  │ of dim_date │  │ of dim_date │
   └─────────────┘  └─────────────┘  └─────────────┘
   
   ✅ JOINs are always local (every node has the full table)
   ❌ Wastes storage (N copies)
   ❌ SLOW inserts/updates (must update N copies)

3. EVEN Distribution
   → Rows distributed round-robin across all nodes
   → Use when no good distribution key exists
   
   DISTSTYLE EVEN
   
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   Node 1    │  │   Node 2    │  │   Node 3    │
   │ Row 1,4,7.. │  │ Row 2,5,8.. │  │ Row 3,6,9.. │
   └─────────────┘  └─────────────┘  └─────────────┘
   
   ✅ Perfectly balanced (no skew)
   ❌ JOINs ALWAYS need network shuffle

4. AUTO Distribution (Default — Recommended)
   → Redshift automatically chooses (ALL for small, EVEN for large)
   → Let Redshift decide unless you know better
   
   DISTSTYLE AUTO
```

### Distribution Cheat Sheet

```
┌──────────────────────────────────────────────────────┐
│  HOW TO CHOOSE DISTRIBUTION STYLE                    │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Small dimension table (< 2M rows)?  → DISTSTYLE ALL │
│                                                      │
│  Large fact table that JOINs on a    → DISTSTYLE KEY  │
│  specific column frequently?           DISTKEY(col)   │
│                                                      │
│  Staging table / no clear JOIN key?  → DISTSTYLE EVEN │
│                                                      │
│  Not sure?                           → DISTSTYLE AUTO │
│                                                      │
│  🔥 PRO RULE: Co-locate fact + largest dimension     │
│     fact_sales DISTKEY(customer_key)                 │
│     dim_customer DISTKEY(customer_key)               │
│     → JOIN without network shuffle!                  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 🔑 Sort Keys — HOW Data is Organized on Disk

Sort keys determine the **physical order** of data on disk. Good sort keys = massive performance improvement.

```
Without Sort Key:
┌────────────────────────────────────────────────────────┐
│ Block 1: 2026-03-15, 2025-01-02, 2026-06-01, 2024-... │
│ Block 2: 2025-07-20, 2026-01-15, 2024-12-31, 2025-... │
│ Block 3: 2026-06-02, 2025-03-10, 2024-06-15, 2026-... │
└────────────────────────────────────────────────────────┘
Query: WHERE date = '2026-06-01' → Must scan ALL blocks 🐌

With Sort Key on date:
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│ Block 1:            │  │ Block 2:            │  │ Block 3:            │
│ 2024-01-01          │  │ 2025-01-01          │  │ 2026-01-01          │
│ 2024-01-02          │  │ 2025-01-02          │  │ 2026-01-02          │
│ ...                 │  │ ...                 │  │ ...                 │
│ 2024-12-31          │  │ 2025-12-31          │  │ 2026-06-02          │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
Query: WHERE date = '2026-06-01' → SKIP Block 1 & 2, scan only Block 3 ⚡
```

### Zone Maps — How Redshift Skips Blocks

```
Redshift stores MIN/MAX metadata for each 1MB block:

Block 1: min_date=2024-01-01, max_date=2024-12-31
Block 2: min_date=2025-01-01, max_date=2025-12-31
Block 3: min_date=2026-01-01, max_date=2026-06-02

Query: WHERE date = '2026-06-01'
→ Block 1: max=2024-12-31 < 2026-06-01 → SKIP ✅
→ Block 2: max=2025-12-31 < 2026-06-01 → SKIP ✅
→ Block 3: min=2026-01-01 ≤ 2026-06-01 ≤ max=2026-06-02 → SCAN 🔍

Result: Read 1 block instead of 3 → 3x faster!
With billions of rows, this means skipping 95%+ of data.
```

### Sort Key Types

```
COMPOUND SORT KEY (Default)
→ Sorts by column1 FIRST, then column2 within column1, etc.
→ Like a phone book: sorted by last name, then first name

CREATE TABLE fact_sales (
    ...
) COMPOUND SORTKEY (date_key, customer_key);

                Sorted data on disk:
                date_key | customer_key | amount
                20260101 | 1001         | 50.00
                20260101 | 1002         | 75.00  ← Within same date,
                20260101 | 1003         | 30.00     sorted by customer
                20260102 | 1001         | 90.00
                20260102 | 1005         | 45.00

✅ Perfect for: WHERE date_key = X (first column)
✅ Good for:    WHERE date_key = X AND customer_key = Y
❌ Useless for: WHERE customer_key = Y (skips first column!)
   (Like looking up "John" in a phone book sorted by last name)


INTERLEAVED SORT KEY
→ Treats all sort columns EQUALLY — no "first column advantage"
→ Works well when queries filter on different columns

CREATE TABLE fact_sales (
    ...
) INTERLEAVED SORTKEY (date_key, customer_key, product_key);

✅ WHERE date_key = X                    → Works
✅ WHERE customer_key = Y                → Works  
✅ WHERE product_key = Z                 → Works
✅ WHERE date_key = X AND product_key = Z → Works (any combination)
❌ VACUUM REINDEX is expensive
❌ Less effective than compound for single-column filters
❌ Being phased out — use AUTO sort key instead
```

### Sort Key Best Practices

```
┌──────────────────────────────────────────────────────┐
│  SORT KEY SELECTION RULES                            │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. Put your most common WHERE/filter column FIRST   │
│     → Almost always: date_key (time-series queries)  │
│                                                      │
│  2. Columns in ORDER BY and GROUP BY benefit too     │
│                                                      │
│  3. JOIN columns get some benefit (merge join)       │
│                                                      │
│  4. Don't use more than 4-5 sort key columns         │
│     → Diminishing returns after that                 │
│                                                      │
│  5. High-cardinality columns first (date > status)   │
│                                                      │
│  🔥 MOST COMMON PATTERN:                             │
│     fact_sales SORTKEY (date_key, customer_key)      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 🗜️ Compression (Encoding) — Shrink Your Data

Redshift is columnar → same data types stored together → compresses incredibly well.

```
Column-oriented data compresses brilliantly:

country column: ["India","India","India","India","USA","USA","USA","India"]
→ Run-Length Encoding: [("India",4),("USA",3),("India",1)]
→ Compressed from 8 values to 3 entries = 62% reduction!

Compression benefits:
✅ Less storage = lower cost
✅ Less data to read from disk = faster queries
✅ Less data to transfer between nodes = faster network
✅ Redshift can process compressed data directly in many cases
```

### Encoding Types

```
┌──────────────────┬─────────────────────────────────────────┐
│ Encoding         │ Best For                                │
├──────────────────┼─────────────────────────────────────────┤
│ RAW              │ No compression (sort key columns)       │
│ AZ64             │ Numeric/date types (DEFAULT — best)     │
│ ZSTD             │ VARCHAR/CHAR (general purpose — best)   │
│ LZO              │ Very long strings                       │
│ BYTEDICT         │ Low-cardinality (<256 distinct values)  │
│ DELTA            │ Sequential numbers (timestamps, IDs)    │
│ RUNLENGTH        │ Sorted columns with many repeats        │
│ MOSTLY8/16/32    │ Integers where most values are small    │
└──────────────────┴─────────────────────────────────────────┘

🔥 PRO TIP: Use ANALYZE COMPRESSION to let Redshift recommend encodings:

ANALYZE COMPRESSION fact_sales;

Output:
Column          | Encoding  | Est. Reduction
----------------+-----------+---------------
date_key        | az64      | 88%
customer_key    | az64      | 75%
product_name    | zstd      | 65%
city            | bytedict  | 92%
amount          | az64      | 80%
```

---

## 🌊 Redshift Spectrum — Query S3 Without Loading

```
Traditional Redshift:
  S3 → COPY → Redshift Tables → Query

Redshift Spectrum:
  S3 → Query DIRECTLY (no loading needed!)

┌─────────────────────┐
│   Redshift Cluster   │
│   (your tables +     │
│    hot data)         │
│   ┌────────────────┐ │         ┌──────────────────┐
│   │  fact_sales    │ │         │  Amazon S3       │
│   │  (2026 data)   │ │         │  (cold data)     │
│   │  LOCAL tables  │ │         │  ┌──────────────┐│
│   └────────────────┘ │         │  │fact_sales_   ││
│                      │ Spectrum│  │archive       ││
│   External schema ───┼────────►│  │(2020-2025)   ││
│                      │ queries │  │Parquet format││
│                      │         │  └──────────────┘│
└─────────────────────┘         └──────────────────┘

-- Create external schema pointing to S3
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'my_spectrum_db'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;

-- Create external table (data stays in S3)
CREATE EXTERNAL TABLE spectrum_schema.sales_archive (
    date_key     INT,
    customer_key INT,
    product_key  INT,
    amount       DECIMAL(12,2)
)
STORED AS PARQUET
LOCATION 's3://my-data-lake/sales/archive/';

-- Query across local AND S3 data seamlessly!
SELECT year, SUM(amount) 
FROM (
    SELECT date_key, amount FROM fact_sales           -- Local (2026)
    UNION ALL
    SELECT date_key, amount FROM spectrum_schema.sales_archive  -- S3 (2020-2025)
) combined
JOIN dim_date d ON combined.date_key = d.date_key
GROUP BY year
ORDER BY year;

Benefits:
✅ Query petabytes in S3 without loading
✅ Keep hot data local, cold data in cheap S3
✅ Supports Parquet, ORC, JSON, CSV, Avro
✅ Integrates with AWS Glue Data Catalog
```

---

## ⚡ Loading Data — The COPY Command

The `COPY` command is the **fastest way** to load data into Redshift (NOT INSERT statements).

```sql
-- ❌ WRONG: Loading data row by row (painfully slow)
INSERT INTO fact_sales VALUES (1, 20260601, 1001, 501, 100, 999.00);
INSERT INTO fact_sales VALUES (2, 20260601, 1002, 502, 50,  499.00);
-- 1 million rows = 1 million round trips = hours 🐌

-- ✅ RIGHT: COPY from S3 (massively parallel)
COPY fact_sales
FROM 's3://my-bucket/sales/2026/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftLoadRole'
FORMAT AS PARQUET;

-- All nodes load in parallel from S3 → millions of rows in seconds ⚡
```

### COPY Best Practices

```
1. SPLIT FILES TO MATCH SLICES
   → If you have 8 slices, split into 8 (or multiples of 8) files
   → Each slice loads one file in parallel
   
   s3://bucket/sales/part-001.parquet
   s3://bucket/sales/part-002.parquet
   s3://bucket/sales/part-003.parquet   → 8 files = 8 slices loading
   ...                                    simultaneously
   s3://bucket/sales/part-008.parquet

2. USE COMPRESSED FILES
   COPY fact_sales FROM 's3://bucket/sales/'
   GZIP;                    -- or LZOP, BZIP2, ZSTD

3. USE MANIFEST FOR SPECIFIC FILES
   COPY fact_sales FROM 's3://bucket/manifest.json'
   MANIFEST;                -- Only load listed files

4. HANDLE ERRORS GRACEFULLY  
   COPY fact_sales FROM 's3://bucket/sales/'
   MAXERROR 100             -- Allow up to 100 bad rows
   ACCEPTINVCHARS '_';      -- Replace invalid UTF-8 chars

5. CHECK LOAD ERRORS
   SELECT * FROM stl_load_errors ORDER BY starttime DESC LIMIT 10;
```

---

## 🔧 Workload Management (WLM) & Concurrency

```
Problem: 
  → Analyst A runs a 2-hour report → hogs all resources
  → Dashboard B needs quick 5-second queries → stuck waiting
  
Solution: WLM Queues — Separate fast queries from heavy ones

┌──────────────────────────────────────────────────────┐
│  WLM CONFIGURATION                                   │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Queue 1: "Dashboard" (Priority: HIGH)               │
│  → Max concurrency: 15                               │
│  → Memory: 40%                                       │
│  → Timeout: 60 seconds                               │
│  → Match: User group 'dashboard_users'               │
│                                                      │
│  Queue 2: "Analysts" (Priority: MEDIUM)              │
│  → Max concurrency: 5                                │
│  → Memory: 35%                                       │
│  → Timeout: 30 minutes                               │
│  → Match: User group 'analysts'                      │
│                                                      │
│  Queue 3: "ETL" (Priority: LOW)                      │
│  → Max concurrency: 3                                │
│  → Memory: 25%                                       │
│  → Timeout: 2 hours                                  │
│  → Match: User group 'etl_service'                   │
│                                                      │
└──────────────────────────────────────────────────────┘

Concurrency Scaling:
→ When queues are full, Redshift spins up ADDITIONAL clusters
→ Handles burst traffic without slowing anyone down
→ First few hours free per day, then billed per second
```

---

## 🔍 Query Optimization — Making Redshift Fly

### EXPLAIN — Read the Execution Plan

```sql
EXPLAIN
SELECT p.category, SUM(f.amount) AS revenue
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
WHERE f.date_key BETWEEN 20260101 AND 20260331
GROUP BY p.category;

-- Key things to look for in the plan:
--
-- DS_DIST_NONE        → No redistribution needed (data co-located) ✅ BEST
-- DS_DIST_ALL_NONE    → Small table broadcast (DISTSTYLE ALL) ✅ GOOD
-- DS_BCAST_INNER      → Inner table broadcast to all nodes ⚠️ OK for small
-- DS_DIST_BOTH        → BOTH tables redistributed → 🔴 VERY SLOW
-- DS_DIST_INNER       → Inner table redistributed → ⚠️ Check DISTKEY
-- DS_DIST_OUTER       → Outer table redistributed → ⚠️ Check DISTKEY
```

### Top Performance Anti-Patterns

```
┌──────────────────────────────────────────────────────────────┐
│  🔴 ANTI-PATTERNS (Things that kill Redshift performance)    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ❌ SELECT *                                                 │
│     → Column-store reads EVERY column = no benefit           │
│     → SELECT only the columns you need                       │
│                                                              │
│  ❌ Single-row INSERT statements                             │
│     → Use COPY command for bulk loads                        │
│                                                              │
│  ❌ Wrong DISTKEY causing DS_DIST_BOTH                       │
│     → Check EXPLAIN for redistribution                       │
│     → Co-locate JOINed tables on the same DISTKEY            │
│                                                              │
│  ❌ Missing SORT KEY on filter columns                       │
│     → Queries scan all blocks instead of zone-map skipping   │
│                                                              │
│  ❌ Large VARCHAR when SMALLINT works                        │
│     → VARCHAR(1000) for a "Y/N" column wastes memory         │
│     → Use the smallest data type that fits                   │
│                                                              │
│  ❌ Not running VACUUM and ANALYZE                           │
│     → Deleted rows aren't reclaimed until VACUUM             │
│     → Statistics become stale → bad query plans              │
│                                                              │
│  ❌ Cross-database queries without Spectrum                  │
│     → Use Spectrum for S3 data, not federated queries        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Essential Maintenance Commands

```sql
-- Reclaim space from deleted rows + re-sort data
VACUUM FULL fact_sales;

-- Lightweight vacuum (reclaim space only, no re-sort)
VACUUM DELETE ONLY fact_sales;

-- Update statistics for the query optimizer
ANALYZE fact_sales;

-- Check table design — get recommendations!
SELECT * FROM svv_table_info WHERE "table" = 'fact_sales';

-- Find queries that are slow
SELECT query, elapsed/1000000 as seconds, substring(querytxt,1,100)
FROM stl_query
WHERE elapsed > 10000000  -- > 10 seconds
ORDER BY elapsed DESC
LIMIT 20;

-- Check for data skew (uneven distribution)
SELECT slice, COUNT(*) as rows
FROM stl_scan
WHERE query = 12345
GROUP BY slice
ORDER BY slice;
-- If one slice has 10x more rows → bad DISTKEY → skew!
```

---

## 🔐 Security Features

```
┌──────────────────────────────────────────────────────┐
│  REDSHIFT SECURITY LAYERS                            │
├──────────────────────────────────────────────────────┤
│                                                      │
│  🔒 Network: VPC, Security Groups, Private subnets   │
│  🔑 Authentication: IAM, DB users, SAML/SSO          │
│  👥 Authorization: GRANT/REVOKE on schemas/tables    │
│  🔐 Encryption at Rest: AWS KMS or HSM               │
│  🔐 Encryption in Transit: SSL/TLS connections       │
│  📋 Audit: System tables (stl_query, stl_connection)│
│  🛡️ Column-Level: GRANT SELECT(col1,col2) ON table  │
│  🏷️ Row-Level Security: Create RLS policies          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

```sql
-- Create users with different access levels
CREATE USER analyst_user PASSWORD 'S3cur3P@ss!';
CREATE USER dashboard_svc PASSWORD 'D@shB0ard!';

-- Create schemas for separation
CREATE SCHEMA sales_mart;
CREATE SCHEMA hr_mart;

-- Grant access
GRANT USAGE ON SCHEMA sales_mart TO analyst_user;
GRANT SELECT ON ALL TABLES IN SCHEMA sales_mart TO analyst_user;

-- Row-Level Security (RLS)
CREATE RLS POLICY region_filter
USING (region = current_setting('app.user_region'));

ATTACH RLS POLICY region_filter ON fact_sales TO analyst_user;
-- Analysts only see data for their region!
```

---

## 🆚 Redshift vs Others — Quick Comparison

```
┌──────────────┬────────────────┬────────────────┬────────────────┐
│ Feature      │ Redshift       │ BigQuery       │ Snowflake      │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ Model        │ Provisioned/   │ Serverless     │ Separated      │
│              │ Serverless     │ (fully)        │ compute+storage│
│ Pricing      │ Per node/hour  │ Per TB scanned │ Per credit/sec │
│              │ + storage      │ + storage      │ + storage      │
│ Storage      │ Columnar       │ Columnar       │ Micro-partition│
│              │ (local+S3)     │ (Capacitor)    │ (S3-based)     │
│ Concurrency  │ WLM queues +   │ Unlimited      │ Multi-cluster  │
│              │ scaling        │ (slots)        │ warehouses     │
│ Ecosystem    │ AWS native     │ GCP native     │ Cloud agnostic │
│ Semi-struct. │ SUPER type     │ Native JSON    │ VARIANT type   │
│ ML           │ Redshift ML    │ BigQuery ML    │ Snowpark       │
│ Best For     │ AWS-heavy orgs │ GCP / ad-hoc   │ Multi-cloud    │
│              │                │ analytics      │ flexibility    │
└──────────────┴────────────────┴────────────────┴────────────────┘
```

---

## 📝 Complete Table DDL Example

```sql
-- Full production-ready Redshift table
CREATE TABLE fact_sales (
    sale_key        BIGINT        IDENTITY(1,1),
    date_key        INT           NOT NULL     ENCODE az64,
    customer_key    INT           NOT NULL     ENCODE az64,
    product_key     INT           NOT NULL     ENCODE az64,
    store_key       INT           NOT NULL     ENCODE az64,
    promo_key       INT           NOT NULL     ENCODE az64,
    order_number    VARCHAR(20)   NOT NULL     ENCODE zstd,
    quantity_sold   SMALLINT      NOT NULL     ENCODE az64,
    unit_price      DECIMAL(10,2) NOT NULL     ENCODE az64,
    discount_pct    DECIMAL(5,2)  DEFAULT 0    ENCODE az64,
    net_amount      DECIMAL(12,2) NOT NULL     ENCODE az64,
    tax_amount      DECIMAL(10,2) NOT NULL     ENCODE az64,
    total_amount    DECIMAL(12,2) NOT NULL     ENCODE az64,
    cost_amount     DECIMAL(12,2) NOT NULL     ENCODE az64,
    profit_amount   DECIMAL(12,2) NOT NULL     ENCODE az64
)
DISTSTYLE KEY
DISTKEY (customer_key)
COMPOUND SORTKEY (date_key, customer_key);

-- Small dimension: distribute to ALL nodes
CREATE TABLE dim_product (
    product_key   INT           PRIMARY KEY  ENCODE az64,
    product_id    VARCHAR(20)   NOT NULL     ENCODE zstd,
    product_name  VARCHAR(200)  NOT NULL     ENCODE zstd,
    category      VARCHAR(50)   NOT NULL     ENCODE bytedict,
    subcategory   VARCHAR(50)   NOT NULL     ENCODE bytedict,
    brand         VARCHAR(50)   NOT NULL     ENCODE bytedict,
    unit_cost     DECIMAL(10,2) NOT NULL     ENCODE az64,
    is_current    CHAR(1)       DEFAULT 'Y'  ENCODE raw
)
DISTSTYLE ALL
SORTKEY (product_key);
```

---

## 🧪 Quick Knowledge Check

```
Q1: Why is COPY faster than INSERT in Redshift?
A1: COPY loads in parallel across all slices from S3.
    INSERT is single-row, single-node, sequential.

Q2: What happens if you use DISTKEY on a skewed column (e.g., country)?
A2: One node gets most data (e.g., "USA" = 80%), others sit idle.
    Query parallelism is destroyed. Use EVEN instead.

Q3: When does DS_DIST_BOTH appear in EXPLAIN?
A3: When BOTH tables in a JOIN need to be redistributed across nodes.
    Fix: set matching DISTKEYs on the JOIN column.

Q4: What is Redshift Spectrum?
A4: A feature that lets you query data in S3 directly without loading it.
    Great for cold/archive data. Supports Parquet, ORC, CSV, JSON.

Q5: Compound vs Interleaved sort key?
A5: Compound: first column gets most benefit (like phone book).
    Interleaved: all columns benefit equally, but maintenance is expensive.
```

---

## 🗺️ Chapter Summary

```
┌────────────────────────────────────────────────────────┐
│  AMAZON REDSHIFT — KEY TAKEAWAYS                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ✅ MPP + Columnar architecture (PostgreSQL-based)     │
│  ✅ Leader node + Compute nodes + Slices                │
│  ✅ DISTKEY: co-locate JOINed data on same node        │
│  ✅ SORTKEY: enable zone-map block skipping            │
│  ✅ COPY command: parallel bulk loading from S3        │
│  ✅ Spectrum: query S3 data without loading            │
│  ✅ Compression: az64 for numbers, zstd for strings    │
│  ✅ WLM: separate fast queries from heavy ones         │
│  ✅ RA3 nodes: separate compute from storage           │
│  ✅ Serverless: zero-admin option for getting started  │
│                                                        │
│  🔥 INTERVIEW ESSENTIALS:                              │
│     Distribution styles, Sort keys, COPY vs INSERT,    │
│     Zone maps, Spectrum, Concurrency Scaling           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [6.3 — Google BigQuery →](./03-BigQuery.md)

---

*"Redshift didn't just democratize data warehousing — it obliterated the price barrier."*
