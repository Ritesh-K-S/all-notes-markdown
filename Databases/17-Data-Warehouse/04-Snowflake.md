# 6.4 — Snowflake — The Cloud Data Platform 🟡🔥

> **"Snowflake is the Switzerland of data warehousing — neutral (multi-cloud), precise (performance), and everyone wants to put their data there."**

> **Level:** 🟡 Intermediate | 🔥 High Demand  
> **Time to Master:** ~4-5 hours  
> **Prerequisites:** Chapter 6.1 (Data Warehouse Concepts)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand Snowflake's **unique multi-cluster shared data architecture**
- Master **Virtual Warehouses** — the compute engine you control
- Use **Time Travel** and **Zero-Copy Cloning** — features that redefine what's possible
- Design tables with **micro-partitions** and **clustering keys**
- Share data instantly with **Secure Data Sharing** (no data movement!)
- Handle semi-structured data (JSON, Avro, Parquet) natively with **VARIANT**
- Know when Snowflake beats Redshift and BigQuery

---

## 🧠 What is Snowflake?

```
Snowflake = Cloud-native data platform with separated compute & storage

Key Facts:
┌──────────────────────────────────────────────────────────┐
│ • Built FROM SCRATCH for the cloud (2012, launched 2014) │
│ • NOT based on any existing database (unlike Redshift)   │
│ • Runs on AWS, Azure, AND GCP — your choice              │
│ • Separates compute, storage, and services completely    │
│ • Near-zero administration (no indexes, no VACUUM)       │
│ • IPO in 2020: largest software IPO in history ($3.4B)   │
│ • Used by: Capital One, Adobe, Siemens, DoorDash, NBC    │
│ • Standard SQL (ANSI compliant)                          │
└──────────────────────────────────────────────────────────┘

Why the name "Snowflake"?
→ Every snowflake is unique — every data workload is unique
→ The architecture adapts to each workload independently
→ Also: data = "cold" storage, snowflakes = data crystals (marketing poetry 😄)
```

---

## 🏗️ Architecture — The Three-Layer Design

This is what makes Snowflake special. **Three independent layers** that scale separately.

```
┌─────────────────────────────────────────────────────────────────┐
│                    SNOWFLAKE ARCHITECTURE                        │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  ☁️ LAYER 3: CLOUD SERVICES (The "Brain")                 │  │
│  │                                                           │  │
│  │  • Authentication & Access Control                        │  │
│  │  • Query parsing, optimization & compilation              │  │
│  │  • Metadata management (micro-partition catalog)          │  │
│  │  • Infrastructure management                              │  │
│  │  • Transaction management                                 │  │
│  │  • Result caching (24-hour query result cache)            │  │
│  │                                                           │  │
│  │  💡 Always running, no cost for basic operations          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  ⚡ LAYER 2: VIRTUAL WAREHOUSES (The "Muscle")            │  │
│  │                                                           │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐             │  │
│  │  │ Warehouse │  │ Warehouse │  │ Warehouse │             │  │
│  │  │  "ETL"    │  │ "BI_DASH" │  │ "ANALYST" │             │  │
│  │  │  XL       │  │  Medium   │  │  Small    │             │  │
│  │  │  $$$      │  │  $$       │  │  $        │             │  │
│  │  └───────────┘  └───────────┘  └───────────┘             │  │
│  │                                                           │  │
│  │  • Each warehouse = independent compute cluster           │  │
│  │  • Can create UNLIMITED warehouses                        │  │
│  │  • Scale up/down INSTANTLY (XS → 4XL)                     │  │
│  │  • Auto-suspend when idle (save $$$)                      │  │
│  │  • Auto-resume on query (no cold start penalty)           │  │
│  │  • Warehouses DON'T compete for resources!                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│                    All warehouses share                          │
│                    the SAME data layer                           │
│                           ▼                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  📦 LAYER 1: STORAGE (The "Memory")                       │  │
│  │                                                           │  │
│  │  • Data stored as micro-partitions in cloud storage       │  │
│  │  • AWS S3 / Azure Blob / GCP Cloud Storage                │  │
│  │  • Columnar format, compressed, encrypted                 │  │
│  │  • Metadata tracks min/max per micro-partition            │  │
│  │  • $23/TB/month (on-demand) — very cheap                  │  │
│  │  • Storage scales INDEPENDENTLY from compute              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why This Architecture Is Revolutionary

```
Traditional Warehouse (Redshift, Teradata):
┌─────────────────────────────────────────────┐
│  Compute + Storage = TIED TOGETHER           │
│  Want more storage? → Buy more nodes         │
│  Want more compute? → Buy more nodes         │
│  ETL running? → Dashboard queries slow down  │
│  3 AM batch job? → Paying for compute 24/7   │
└─────────────────────────────────────────────┘

Snowflake:
┌─────────────────────────────────────────────┐
│  Compute + Storage = INDEPENDENT             │
│  Want more storage? → Just store more (auto) │
│  Want more compute? → Spin up bigger WH      │
│  ETL running? → Dashboard uses separate WH   │
│  3 AM batch job? → WH auto-suspends after    │
│  No users? → $0 compute cost                 │
└─────────────────────────────────────────────┘
```

---

## ⚡ Virtual Warehouses — Deep Dive

Virtual Warehouses are the **compute engines** in Snowflake. Understanding them is key to cost control.

```
┌───────────────────────────────────────────────────────────────┐
│  WAREHOUSE SIZES                                              │
├──────────┬──────────────┬─────────────────────────────────────┤
│ Size     │ Credits/Hour │ Servers (approx)                    │
├──────────┼──────────────┼─────────────────────────────────────┤
│ X-Small  │      1       │ 1 server                            │
│ Small    │      2       │ 2 servers                           │
│ Medium   │      4       │ 4 servers                           │
│ Large    │      8       │ 8 servers                           │
│ X-Large  │     16       │ 16 servers                          │
│ 2X-Large │     32       │ 32 servers                          │
│ 3X-Large │     64       │ 64 servers                          │
│ 4X-Large │    128       │ 128 servers                         │
│ 5X-Large │    256       │ 256 servers                         │
│ 6X-Large │    512       │ 512 servers (!!!)                   │
├──────────┴──────────────┴─────────────────────────────────────┤
│  Each size DOUBLES the previous                               │
│  1 credit ≈ $2-4 depending on edition & cloud provider        │
│  Doubling size = 2x cost but often 2x speed → same total cost│
└───────────────────────────────────────────────────────────────┘
```

```sql
-- Create warehouses for different workloads
CREATE WAREHOUSE etl_warehouse
    WAREHOUSE_SIZE = 'X-LARGE'
    AUTO_SUSPEND = 300            -- Suspend after 5 min idle
    AUTO_RESUME = TRUE            -- Auto-start on query
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3         -- Multi-cluster: scale out!
    SCALING_POLICY = 'STANDARD';

CREATE WAREHOUSE dashboard_warehouse
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 60             -- Suspend after 1 min idle
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 10        -- Handle dashboard traffic spikes!
    SCALING_POLICY = 'ECONOMY';

CREATE WAREHOUSE analyst_warehouse
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE;

-- Switch warehouse for a session
USE WAREHOUSE analyst_warehouse;

-- Resize on the fly (no downtime!)
ALTER WAREHOUSE etl_warehouse SET WAREHOUSE_SIZE = '2X-LARGE';
-- After ETL completes:
ALTER WAREHOUSE etl_warehouse SET WAREHOUSE_SIZE = 'X-LARGE';
```

### Multi-Cluster Warehouses

```
Problem: 50 analysts all run queries at 10 AM → one warehouse is overwhelmed

Solution: Multi-cluster warehouse automatically adds clusters

10 AM: 5 queries → 1 cluster handles it
┌───────────┐
│ Cluster 1 │  ← Handles all 5 queries
└───────────┘

10:15 AM: 50 queries → 3 clusters auto-scale
┌───────────┐ ┌───────────┐ ┌───────────┐
│ Cluster 1 │ │ Cluster 2 │ │ Cluster 3 │  ← Load balanced!
└───────────┘ └───────────┘ └───────────┘

11 AM: 3 queries → scales back to 1 cluster
┌───────────┐
│ Cluster 1 │  ← Auto-scaled down to save cost
└───────────┘

Scaling Policies:
• STANDARD: Add cluster when queries start queueing (responsive)
• ECONOMY:  Add cluster only when significant queueing (cost-saving)
```

---

## 📦 Micro-Partitions — Snowflake's Storage Secret

Snowflake **doesn't use traditional indexes**. Instead, it uses micro-partitions with metadata pruning.

```
Traditional DB: CREATE INDEX → you manage it
Snowflake:      Micro-partitions → automatic, no management!

How it works:
┌──────────────────────────────────────────────────────────────┐
│  Every table is automatically divided into MICRO-PARTITIONS  │
│  Each micro-partition:                                       │
│  • Contains 50-500 MB of uncompressed data                   │
│  • Stored in columnar format                                 │
│  • Compressed independently                                  │
│  • Has metadata: MIN, MAX, COUNT, NULL count per column      │
│  • Is IMMUTABLE (never modified, only replaced)              │
└──────────────────────────────────────────────────────────────┘

fact_sales table (1 TB total):
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐     ┌────────┐
│ MP-1   │ │ MP-2   │ │ MP-3   │ │ MP-4   │ ... │ MP-2000│
│ 500MB  │ │ 500MB  │ │ 500MB  │ │ 500MB  │     │ 500MB  │
│        │ │        │ │        │ │        │     │        │
│min_dt: │ │min_dt: │ │min_dt: │ │min_dt: │     │min_dt: │
│2026-01 │ │2026-01 │ │2026-02 │ │2026-03 │     │2026-06 │
│max_dt: │ │max_dt: │ │max_dt: │ │max_dt: │     │max_dt: │
│2026-01 │ │2026-01 │ │2026-02 │ │2026-03 │     │2026-06 │
└────────┘ └────────┘ └────────┘ └────────┘     └────────┘

Query: WHERE order_date = '2026-03-15'
Step 1: Check metadata of each micro-partition
Step 2: MP-1 (Jan) → SKIP, MP-2 (Jan) → SKIP, MP-3 (Feb) → SKIP
Step 3: MP-4 (Mar) → min ≤ 2026-03-15 ≤ max → SCAN this one!
Step 4: Skipped ~95% of data without ANY index! 

This is called PARTITION PRUNING.
```

### Clustering Keys — Improve Pruning

```
By default, data is micro-partitioned in insertion order.
This can lead to OVERLAPPING partitions:

WITHOUT clustering:
MP-1: dates=[Jan, Mar, Jun]     ← Overlapping ranges!
MP-2: dates=[Feb, Apr, May]
MP-3: dates=[Jan, Jun, Mar]

Query WHERE date = 'Mar' → Must scan ALL partitions (no pruning possible)

WITH CLUSTER BY (order_date):
MP-1: dates=[Jan, Jan, Feb]     ← Clean ranges!
MP-2: dates=[Mar, Mar, Apr]
MP-3: dates=[May, Jun, Jun]

Query WHERE date = 'Mar' → Only scan MP-2 → 66% pruned!
```

```sql
-- Add clustering key to a table
ALTER TABLE fact_sales CLUSTER BY (order_date, category);

-- Clustering is AUTOMATIC — Snowflake reclusters in the background
-- No manual VACUUM or REINDEX needed!

-- Check clustering quality
SELECT SYSTEM$CLUSTERING_INFORMATION('fact_sales', '(order_date)');

-- Returns:
-- {
--   "cluster_by_keys": "LINEAR(order_date)",
--   "total_partition_count": 2000,
--   "total_constant_partition_count": 800,  ← Partitions with single value
--   "average_overlaps": 1.5,                ← Lower = better (ideal: 0)
--   "average_depth": 2.1                    ← Lower = better (ideal: 1)
-- }
```

### When to Use Clustering Keys

```
┌──────────────────────────────────────────────────────────────┐
│  CLUSTERING KEY GUIDELINES                                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ USE clustering when:                                     │
│  → Table > 1 TB                                             │
│  → Queries frequently filter on specific columns             │
│  → Pruning ratio is poor (check with SYSTEM$CLUSTERING_INFO) │
│                                                              │
│  ❌ DON'T cluster when:                                      │
│  → Table < 1 TB (overhead not worth it)                      │
│  → Data is already well-ordered naturally                    │
│  → No clear filter pattern in queries                        │
│                                                              │
│  Column Order (max 4):                                       │
│  → Put lowest cardinality first: category (10) before        │
│    customer_id (1M)                                          │
│  → Date columns are great candidates                         │
│  → Commonly filtered columns                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## ⏰ Time Travel — Query & Restore Past Data

One of Snowflake's most loved features. Access historical data after changes.

```
┌──────────────────────────────────────────────────────────────┐
│  TIME TRAVEL — How It Works                                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  When you INSERT, UPDATE, or DELETE:                         │
│  → Snowflake creates NEW micro-partitions                    │
│  → OLD micro-partitions are KEPT (not deleted)               │
│  → You can query the table as it was at ANY point in time    │
│                                                              │
│  Retention Period:                                           │
│  → Standard Edition: up to 1 day                            │
│  → Enterprise Edition: up to 90 days                        │
│  → Business Critical: up to 90 days                         │
│                                                              │
│  After retention expires → data moves to FAIL-SAFE          │
│  → 7 additional days (Snowflake support can recover)         │
│  → Not user-accessible                                       │
│                                                              │
│  Timeline:                                                   │
│  ──────────────────────────────────────────────►             │
│  │ Current │ Time Travel │  Fail-Safe  │  Purged │           │
│  │  Data   │ (1-90 days) │  (7 days)   │  Gone   │           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```sql
-- Query table as it was 1 hour ago
SELECT * FROM fact_sales AT(OFFSET => -3600);  -- 3600 seconds ago

-- Query table as it was at a specific timestamp
SELECT * FROM fact_sales AT(TIMESTAMP => '2026-06-01 10:00:00'::TIMESTAMP);

-- Query table before a specific statement was executed
SELECT * FROM fact_sales BEFORE(STATEMENT => '8e5d0ca9-005e-44e6-b858-a8f5b37c5726');

-- ⭐ UNDO an accidental DELETE!
-- Step 1: Oops! Someone ran:
DELETE FROM fact_sales WHERE year = 2026;  -- 😱 All 2026 data gone!

-- Step 2: Time travel to the rescue!
CREATE TABLE fact_sales_recovered 
    CLONE fact_sales AT(OFFSET => -300);  -- 5 minutes ago

-- Step 3: Restore the data
INSERT INTO fact_sales SELECT * FROM fact_sales_recovered WHERE year = 2026;

-- Or even simpler with UNDROP:
DROP TABLE fact_sales;       -- Oops!
UNDROP TABLE fact_sales;     -- 😅 Recovered in 1 second!

-- Works for schemas and databases too:
DROP DATABASE analytics;
UNDROP DATABASE analytics;   -- Full database recovered!
```

---

## 🐑 Zero-Copy Cloning — Instant Copies

This is MAGIC. Create full copies of tables, schemas, or entire databases **instantly** with **zero storage cost** initially.

```
Traditional approach:
CREATE TABLE fact_sales_dev AS SELECT * FROM fact_sales;
→ Time: 30 minutes (copies 1 TB of data)
→ Cost: DOUBLES your storage ($23/TB/month extra)

Snowflake clone:
CREATE TABLE fact_sales_dev CLONE fact_sales;
→ Time: 2 SECONDS (regardless of size!)
→ Cost: $0 initially (no data copied!)

How? Both tables point to the SAME micro-partitions:

BEFORE clone:
┌────────────────┐
│ fact_sales     │──────► [MP-1] [MP-2] [MP-3] [MP-4]
└────────────────┘

AFTER clone:
┌────────────────┐
│ fact_sales     │──────► [MP-1] [MP-2] [MP-3] [MP-4]
└────────────────┘                ▲   ▲   ▲   ▲
┌────────────────┐                │   │   │   │
│ fact_sales_dev │────────────────┘   │   │   │
└────────────────┘ (same micro-partitions, zero storage!)

AFTER modifying the clone:
┌────────────────┐
│ fact_sales     │──────► [MP-1] [MP-2] [MP-3] [MP-4]
└────────────────┘                       ▲        ▲
┌────────────────┐                       │        │
│ fact_sales_dev │──────► [MP-1'] [MP-2']│  [MP-4]│
└────────────────┘   (only CHANGED       │        │
                      partitions are     │shared   │shared
                      new & cost $)
```

```sql
-- Clone a table (instant, free)
CREATE TABLE fact_sales_staging CLONE fact_sales;

-- Clone an entire database! (instant, free)
CREATE DATABASE analytics_dev CLONE analytics;

-- Clone a schema
CREATE SCHEMA staging CLONE production;

-- Clone with time travel (clone from the past!)
CREATE TABLE fact_sales_yesterday CLONE fact_sales
    AT(TIMESTAMP => '2026-06-01 23:59:59'::TIMESTAMP);

-- Use cases:
-- ✅ Development/testing environments (instant, free!)
-- ✅ "What-if" analysis (modify clone, not production)
-- ✅ Backup before risky operations
-- ✅ ML training data snapshots
```

---

## 🔄 Data Sharing — Share Without Copying

Share live data with other Snowflake accounts **without moving a single byte**.

```
Traditional data sharing:
┌──────────┐   Extract   ┌──────────┐   Load    ┌──────────┐
│ Producer │  ──CSV/API── │  S3/FTP  │  ──ETL── │ Consumer │
│ Account  │              │  Staging │          │ Account  │
└──────────┘              └──────────┘          └──────────┘
→ Stale data (hours/days old)
→ Storage cost doubled
→ ETL maintenance
→ Security concerns (data in transit)

Snowflake Data Sharing:
┌──────────┐                          ┌──────────┐
│ Producer │    Shared metadata only   │ Consumer │
│ Account  │ ────────────────────────► │ Account  │
│          │    (no data copied!)      │          │
│ [Data]   │◄─── Consumer queries ────│  Reads   │
│          │     the SAME storage     │  shared  │
│          │                          │  data    │
└──────────┘                          └──────────┘
→ Real-time data (always current!)
→ Zero storage cost for sharing
→ No ETL needed
→ Governed access (producer controls)
```

```sql
-- PRODUCER: Create a share
CREATE SHARE sales_share;
GRANT USAGE ON DATABASE analytics TO SHARE sales_share;
GRANT USAGE ON SCHEMA analytics.public TO SHARE sales_share;
GRANT SELECT ON TABLE analytics.public.fact_sales TO SHARE sales_share;

-- PRODUCER: Add consumer account
ALTER SHARE sales_share ADD ACCOUNT = 'CONSUMER_ACCT_123';

-- CONSUMER: Create database from share (read-only)
CREATE DATABASE shared_sales FROM SHARE producer_acct.sales_share;

-- CONSUMER: Query shared data (always live, always current!)
SELECT category, SUM(amount) 
FROM shared_sales.public.fact_sales 
GROUP BY category;

-- ✅ Consumer always sees latest data
-- ✅ Producer controls what's shared
-- ✅ No data movement, no copies
-- ✅ Revoke access instantly
-- 🔥 This powers the Snowflake Marketplace!
```

---

## 🔀 Semi-Structured Data — VARIANT Type

Snowflake natively handles JSON, Avro, Parquet, ORC, and XML without flattening.

```sql
-- Create a table with VARIANT column
CREATE TABLE raw_events (
    event_id    INT,
    event_date  TIMESTAMP,
    payload     VARIANT        -- Stores any JSON/semi-structured data
);

-- Load JSON data
INSERT INTO raw_events 
SELECT 
    1,
    CURRENT_TIMESTAMP(),
    PARSE_JSON('{
        "user": {"id": 42, "name": "Ritesh", "city": "Mumbai"},
        "action": "purchase",
        "items": [
            {"product": "Phone", "price": 999, "qty": 1},
            {"product": "Case", "price": 49, "qty": 2}
        ],
        "metadata": {"browser": "Chrome", "os": "Windows"}
    }');

-- Query nested JSON using dot notation and bracket notation
SELECT 
    payload:user.name::STRING          AS user_name,
    payload:user.city::STRING          AS city,
    payload:action::STRING             AS action,
    payload:metadata.browser::STRING   AS browser
FROM raw_events;

-- Flatten arrays with LATERAL FLATTEN
SELECT 
    e.event_id,
    e.payload:user.name::STRING         AS user_name,
    item.value:product::STRING          AS product,
    item.value:price::NUMBER            AS price,
    item.value:qty::NUMBER              AS quantity
FROM raw_events e,
LATERAL FLATTEN(input => e.payload:items) item;

-- Result:
-- event_id | user_name | product | price | quantity
-- 1        | Ritesh    | Phone   | 999   | 1
-- 1        | Ritesh    | Case    | 49    | 2

-- 💡 No pre-processing needed! Load raw JSON, query immediately.
-- BigQuery uses STRUCT/ARRAY, Snowflake uses VARIANT — different syntax, same idea.
```

---

## 📊 Stages & Data Loading

```sql
-- Snowflake uses STAGES for data loading (internal or external)

-- 1. INTERNAL STAGE (Snowflake-managed storage)
CREATE STAGE my_internal_stage;
-- Upload using SnowSQL CLI:
-- PUT file://C:/data/sales.csv @my_internal_stage;

-- 2. EXTERNAL STAGE (your S3/Azure/GCS bucket)
CREATE STAGE my_s3_stage
    URL = 's3://my-bucket/sales/'
    CREDENTIALS = (AWS_KEY_ID='...' AWS_SECRET_KEY='...');

-- Or better — use a Storage Integration (no embedded credentials)
CREATE STORAGE INTEGRATION s3_integration
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = 'S3'
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/SnowflakeRole'
    STORAGE_ALLOWED_LOCATIONS = ('s3://my-bucket/');

-- COPY INTO — The primary bulk loading command
COPY INTO fact_sales
FROM @my_s3_stage/2026/
FILE_FORMAT = (TYPE = 'PARQUET')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

-- Load CSV with options
COPY INTO fact_sales
FROM @my_s3_stage
FILE_FORMAT = (
    TYPE = 'CSV'
    SKIP_HEADER = 1
    FIELD_DELIMITER = ','
    NULL_IF = ('NULL', 'null', '')
    ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE
)
ON_ERROR = 'CONTINUE';       -- Skip bad rows
-- Other ON_ERROR options: ABORT_STATEMENT, SKIP_FILE, SKIP_FILE_5%

-- Continuous data loading with SNOWPIPE (serverless, auto-triggered)
CREATE PIPE sales_pipe 
    AUTO_INGEST = TRUE
AS
COPY INTO fact_sales
FROM @my_s3_stage/incoming/
FILE_FORMAT = (TYPE = 'PARQUET');

-- When files land in S3 → Snowpipe detects → auto-loads within minutes!
-- 💡 Snowpipe = serverless, event-driven loading (like a mini-ETL)
```

---

## 🔐 Security — Enterprise-Grade

```
┌──────────────────────────────────────────────────────────────┐
│  SNOWFLAKE SECURITY STACK                                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🌐 Network Security                                        │
│  → Network policies (IP whitelisting)                        │
│  → AWS PrivateLink / Azure Private Link                      │
│  → No public internet exposure option                        │
│                                                              │
│  🔑 Authentication                                          │
│  → Username/password + MFA                                   │
│  → Key pair authentication (RSA 2048-bit)                    │
│  → SSO (SAML 2.0 / OAuth)                                   │
│  → SCIM for user provisioning                                │
│                                                              │
│  👥 Access Control (RBAC)                                    │
│  → Role-based access control                                 │
│  → System roles: ACCOUNTADMIN, SYSADMIN, SECURITYADMIN       │
│  → Custom roles with fine-grained privileges                 │
│  → Role hierarchy (inheritance)                              │
│                                                              │
│  🔐 Encryption                                              │
│  → Always-on encryption at rest (AES-256)                    │
│  → Always-on encryption in transit (TLS 1.2+)               │
│  → Tri-Secret Secure (customer-managed key + Snowflake key)  │
│  → Automatic key rotation                                    │
│                                                              │
│  🛡️ Data Protection                                         │
│  → Column-level masking policies                             │
│  → Row-level access policies                                │
│  → Object tagging (classify sensitive data)                  │
│  → Dynamic data masking                                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```sql
-- Dynamic Data Masking
CREATE MASKING POLICY email_mask AS (val STRING) 
RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'DATA_ENGINEER') THEN val
        ELSE REGEXP_REPLACE(val, '.+@', '***@')
    END;

ALTER TABLE dim_customer MODIFY COLUMN email SET MASKING POLICY email_mask;

-- Admin sees:        ritesh@company.com
-- Analyst sees:      ***@company.com

-- Row Access Policy
CREATE ROW ACCESS POLICY region_policy AS (region_val VARCHAR) 
RETURNS BOOLEAN ->
    CURRENT_ROLE() = 'ADMIN' 
    OR region_val = CURRENT_SESSION()::VARIANT:region::VARCHAR;

ALTER TABLE fact_sales ADD ROW ACCESS POLICY region_policy ON (region);
```

---

## 💰 Cost Management — Credits & Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│  SNOWFLAKE PRICING                                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  COMPUTE: Credits per hour (based on warehouse size)         │
│  ┌──────────┬─────────────────────────────────────┐          │
│  │ Edition  │ Price per Credit                     │          │
│  ├──────────┼─────────────────────────────────────┤          │
│  │ Standard │ ~$2.00                               │          │
│  │ Enterprise│ ~$3.00                              │          │
│  │ Business │ ~$4.00                               │          │
│  │ Critical │                                     │          │
│  └──────────┴─────────────────────────────────────┘          │
│                                                              │
│  STORAGE: Per TB per month                                   │
│  → On-demand: $40/TB/month                                   │
│  → Pre-purchased: $23/TB/month                               │
│                                                              │
│  SERVERLESS FEATURES: Credits used automatically             │
│  → Snowpipe, auto-clustering, materialized views, replication│
│                                                              │
│  🔥 COST OPTIMIZATION:                                       │
│  → Set AUTO_SUSPEND = 60 (1 minute) on warehouses           │
│  → Use RESOURCE MONITORS to set credit budgets               │
│  → Right-size warehouses (don't use XL for small queries)    │
│  → Use result caching (24-hour cache is FREE)                │
│  → Share data instead of copying (zero cost)                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```sql
-- Resource Monitor — prevent cost overruns
CREATE RESOURCE MONITOR monthly_limit
    WITH CREDIT_QUOTA = 5000        -- 5000 credits max per month
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS 
        ON 75 PERCENT DO NOTIFY       -- Alert at 75%
        ON 90 PERCENT DO NOTIFY       -- Alert at 90%
        ON 100 PERCENT DO SUSPEND     -- Suspend warehouses at 100%!
        ON 110 PERCENT DO SUSPEND_IMMEDIATE;

-- Apply to a warehouse
ALTER WAREHOUSE analyst_warehouse SET RESOURCE_MONITOR = monthly_limit;
```

---

## 🆚 Snowflake vs Redshift vs BigQuery — Final Showdown

```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Feature         │ Snowflake       │ Redshift        │ BigQuery        │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Cloud           │ AWS/Azure/GCP   │ AWS only        │ GCP only        │
│ Architecture    │ Separated       │ Provisioned     │ Serverless      │
│                 │ compute+storage │ nodes (RA3      │ (fully managed) │
│                 │                 │ separates)      │                 │
│ Pricing         │ Credits + stor. │ Node/hour + stor│ Per TB scanned  │
│ Concurrency     │ Multi-cluster   │ WLM queues      │ Slots           │
│                 │ warehouses      │                 │                 │
│ Semi-structured │ VARIANT         │ SUPER           │ STRUCT + ARRAY  │
│ Time Travel     │ Up to 90 days   │ Snapshots       │ 7 days          │
│ Cloning         │ ✅ Zero-copy    │ ❌ Full copy     │ ❌ Full copy     │
│ Data Sharing    │ ✅ Native       │ ❌ Export/import │ ✅ Analytics Hub │
│ ML              │ Snowpark        │ Redshift ML     │ BQML (best)     │
│ Ecosystem Lock  │ None (portable) │ AWS ecosystem   │ GCP ecosystem   │
│ Best For        │ Multi-cloud,    │ AWS-native,     │ GCP, ad-hoc     │
│                 │ data sharing,   │ cost-optimized  │ analytics,      │
│                 │ flexibility     │ workloads       │ ML in SQL       │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

---

## 🧪 Quick Knowledge Check

```
Q1: What makes Snowflake's architecture unique?
A1: Three independent layers: Cloud Services (brain), Virtual Warehouses 
    (compute), Storage. Each scales independently.

Q2: How does zero-copy cloning work?
A2: Clone shares the same micro-partitions as the original. Only new/changed 
    data creates new partitions. Instant, initial zero cost.

Q3: What replaces indexes in Snowflake?
A3: Micro-partitions with automatic metadata (min/max per column). 
    Snowflake prunes partitions using this metadata. No manual index management.

Q4: How does data sharing work without copying data?
A4: Producer shares metadata references. Consumer queries the SAME underlying 
    storage. Zero data movement, always up-to-date.

Q5: Multi-cluster warehouse vs scaling up — when to use which?
A5: Scale UP (bigger size): query is slow, needs more compute power.
    Scale OUT (more clusters): many concurrent users, queries are queuing.

Q6: How to prevent cost overruns?
A6: Resource Monitors with credit quotas + auto-suspend warehouses + 
    right-size warehouses + leverage result caching.
```

---

## 🗺️ Chapter Summary

```
┌────────────────────────────────────────────────────────┐
│  SNOWFLAKE — KEY TAKEAWAYS                             │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ✅ Three-layer architecture (services/compute/storage)│
│  ✅ Virtual Warehouses: independent, scalable compute  │
│  ✅ Micro-partitions: automatic, no manual indexing    │
│  ✅ Clustering keys: improve pruning for large tables  │
│  ✅ Time Travel: query/restore past data (up to 90d)   │
│  ✅ Zero-Copy Cloning: instant, free table/DB copies   │
│  ✅ Data Sharing: live data, no movement, no ETL       │
│  ✅ VARIANT: native semi-structured data (JSON/Avro)   │
│  ✅ Snowpipe: serverless continuous data loading       │
│  ✅ Multi-cloud: AWS, Azure, GCP — no lock-in          │
│                                                        │
│  🔥 INTERVIEW ESSENTIALS:                              │
│     Architecture layers, Virtual Warehouses,           │
│     Zero-copy cloning, Time Travel,                    │
│     Data Sharing, Micro-partitions                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [6.5 — Apache Spark SQL & Data Lakehouse →](./05-Spark-SQL-Lakehouse.md)

---

*"Snowflake proved that you don't need to choose between simplicity and power — you can have both."*
