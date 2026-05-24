# Chapter 54 — BigQuery

---

## Table of Contents

- [Overview](#overview)
- [Part 1: BigQuery Fundamentals](#part-1--bigquery-fundamentals)
- [Part 2: Datasets & Tables](#part-2--datasets--tables)
- [Part 3: Loading Data](#part-3--loading-data)
- [Part 4: Querying Data](#part-4--querying-data)
- [Part 5: Partitioning](#part-5--partitioning)
- [Part 6: Clustering](#part-6--clustering)
- [Part 7: Views & Materialized Views](#part-7--views--materialized-views)
- [Part 8: BI Engine](#part-8--bi-engine)
- [Part 9: Streaming Inserts & Storage Write API](#part-9--streaming-inserts--storage-write-api)
- [Part 10: External Tables & Federated Queries](#part-10--external-tables--federated-queries)
- [Part 11: BigQuery ML (BQML)](#part-11--bigquery-ml-bqml)
- [Part 12: Access Control & Security](#part-12--access-control--security)
- [Part 13: Cost Optimization](#part-13--cost-optimization)
- [Part 14: Terraform & bq CLI Reference](#part-14--terraform--bq-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What is BigQuery? (Beginner Explanation)](#what-is-bigquery-beginner-explanation)
- [Console Walkthrough: Using BigQuery](#console-walkthrough-using-bigquery)
- [Additional bq Commands](#additional-bq-commands)
- [What's Next?](#whats-next)

---

## Overview

BigQuery is Google Cloud's fully managed, serverless, petabyte-scale data warehouse designed for analytics. It separates storage from compute, allowing independent scaling — you can store petabytes cheaply and query them with thousands of CPU slots on demand. BigQuery supports standard SQL, real-time streaming, machine learning (BQML), geospatial analytics, and integrates deeply with the Google Cloud ecosystem.

---

## Part 1 — BigQuery Fundamentals

### Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│         BIGQUERY ARCHITECTURE                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  BigQuery separates storage and compute:                            │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  COMPUTE (Dremel Engine)                                   │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │     │
│  │  │ Slot 1   │  │ Slot 2   │  │ Slot N   │              │     │
│  │  │ (vCPU)   │  │ (vCPU)   │  │ (vCPU)   │              │     │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘              │     │
│  │       └──────────────┼──────────────┘                     │     │
│  │                      │ Massively parallel query            │     │
│  └──────────────────────┼───────────────────────────────────┘     │
│                         │                                           │
│  ┌──────────────────────┼───────────────────────────────────┐     │
│  │  STORAGE (Colossus)  │                                     │     │
│  │                      ▼                                     │     │
│  │  ┌──────────────────────────────────────────────┐        │     │
│  │  │ Columnar format (Capacitor)                   │        │     │
│  │  │ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐        │        │     │
│  │  │ │Col A │ │Col B │ │Col C │ │Col D │        │        │     │
│  │  │ │......│ │......│ │......│ │......│        │        │     │
│  │  │ └──────┘ └──────┘ └──────┘ └──────┘        │        │     │
│  │  │ Only scans columns referenced in query       │        │     │
│  │  └──────────────────────────────────────────────┘        │     │
│  │  Automatic compression, encryption, replication          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Key concepts:                                                      │
│  • Slot = unit of compute (vCPU + RAM)                             │
│  • On-demand: auto-allocates up to 2,000 slots                     │
│  • Editions: standard/enterprise/enterprise plus (reserved slots)  │
│  • Jupiter network: 1 Petabit/s between storage and compute       │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP BigQuery | AWS Redshift | Azure Synapse |
|---------|-------------|-------------|--------------|
| Model | Serverless | Cluster-based | Serverless / Dedicated |
| Storage-compute separation | Native | RA3 nodes | Serverless pool |
| Pricing model | Per-query (on-demand) or slots | Per-node-hour | Per-query (serverless) |
| Streaming ingest | Yes (native) | Kinesis Firehose | Event Hubs |
| ML in SQL | BQML | Redshift ML | Synapse ML |
| Geospatial | BigQuery GIS | PostGIS | Spatial types |
| Free tier | 1 TiB query + 10 GiB storage/month | 2-month trial | None |
| Max query size | Unlimited (auto-scales) | Cluster-limited | Unlimited (serverless) |

### Pricing

| Component | On-Demand | Editions (Enterprise) |
|-----------|-----------|----------------------|
| Queries | $6.25/TiB scanned | $0.04/slot-hour (autoscale) |
| Active storage | $0.02/GiB/month | $0.02/GiB/month |
| Long-term storage (90+ days) | $0.01/GiB/month | $0.01/GiB/month |
| Streaming inserts | $0.05/GiB | $0.05/GiB |
| Storage Write API | $0.025/GiB | $0.025/GiB |
| Free tier | 1 TiB queries + 10 GiB storage/month | — |

---

## Part 2 — Datasets & Tables

### Resource Hierarchy

```
┌────────────────────────────────────────────────────────────────────┐
│         BIGQUERY RESOURCE HIERARCHY                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Project                                                            │
│  └── Dataset (location, access control, default expiration)        │
│       ├── Table (schema, partitioning, clustering)                 │
│       │    └── Column (name, type, mode, description)              │
│       ├── View (SQL query, logical view)                           │
│       ├── Materialized View (cached query results)                 │
│       ├── Routine (UDF, stored procedure)                          │
│       └── Model (BQML model)                                      │
│                                                                      │
│  Referencing: `project.dataset.table`                              │
│  Example:     `my-project.analytics.events`                        │
│                                                                      │
│  Table types:                                                       │
│  • Native (managed) — data stored in BigQuery                     │
│  • External — data in GCS, Bigtable, Drive, Cloud SQL             │
│  • Snapshot — point-in-time copy                                   │
│  • Clone — zero-copy reference (copy-on-write)                    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Data Types

| Type | Description | Example |
|------|------------|---------|
| `INT64` | Integer | `42` |
| `FLOAT64` | Floating point | `3.14` |
| `NUMERIC` / `BIGNUMERIC` | Exact decimal | `123.456789` |
| `BOOL` | Boolean | `TRUE` |
| `STRING` | Text | `'hello'` |
| `BYTES` | Binary | `b'\x00\x01'` |
| `DATE` | Date | `DATE '2024-01-15'` |
| `TIME` | Time | `TIME '10:30:00'` |
| `DATETIME` | Date + time (no timezone) | `DATETIME '2024-01-15 10:30:00'` |
| `TIMESTAMP` | Date + time + timezone | `TIMESTAMP '2024-01-15T10:30:00Z'` |
| `GEOGRAPHY` | Geospatial | `ST_GEOGPOINT(-122.4, 37.8)` |
| `JSON` | Semi-structured | `JSON '{"key": "val"}'` |
| `ARRAY` | Repeated values | `[1, 2, 3]` |
| `STRUCT` | Nested record | `STRUCT(1 AS id, 'a' AS name)` |

---

## Part 3 — Loading Data

### Batch Loading

```bash
# Load CSV from GCS
bq load --source_format=CSV \
    --skip_leading_rows=1 \
    --autodetect \
    my_dataset.my_table \
    gs://my-bucket/data/*.csv

# Load JSON (newline-delimited)
bq load --source_format=NEWLINE_DELIMITED_JSON \
    --autodetect \
    my_dataset.events \
    gs://my-bucket/events/2024-01-*.json

# Load Parquet (schema auto-detected from file)
bq load --source_format=PARQUET \
    my_dataset.sales \
    gs://my-bucket/parquet/sales/

# Load Avro
bq load --source_format=AVRO \
    my_dataset.users \
    gs://my-bucket/avro/users.avro

# Load with explicit schema
bq load --source_format=CSV \
    --skip_leading_rows=1 \
    my_dataset.products \
    gs://my-bucket/products.csv \
    id:INTEGER,name:STRING,price:FLOAT,created_at:TIMESTAMP

# Append vs overwrite
bq load --replace \                     # overwrite table
    my_dataset.table gs://bucket/file.csv

bq load --noreplace \                   # append (default)
    my_dataset.table gs://bucket/file.csv
```

### Load Job Options

| Option | Description |
|--------|-------------|
| `--source_format` | CSV, NEWLINE_DELIMITED_JSON, PARQUET, AVRO, ORC |
| `--autodetect` | Auto-detect schema |
| `--skip_leading_rows=N` | Skip header rows (CSV) |
| `--replace` | Overwrite existing data |
| `--max_bad_records=N` | Allow N bad records before failing |
| `--write_disposition` | WRITE_TRUNCATE, WRITE_APPEND, WRITE_EMPTY |
| `--time_partitioning_field` | Partition by this TIMESTAMP/DATE column |
| `--clustering_fields` | Cluster by these columns (comma-separated) |

---

## Part 4 — Querying Data

### Standard SQL

```sql
-- Basic query
SELECT
    user_id,
    event_type,
    COUNT(*) AS event_count
FROM `my-project.analytics.events`
WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY user_id, event_type
ORDER BY event_count DESC
LIMIT 100;

-- Nested/repeated fields (STRUCT and ARRAY)
SELECT
    order_id,
    customer.name AS customer_name,        -- STRUCT access
    item.product_name,                      -- ARRAY element
    item.quantity
FROM `my-project.sales.orders`,
    UNNEST(items) AS item                   -- flatten ARRAY
WHERE customer.country = 'US';

-- Window functions
SELECT
    user_id,
    event_date,
    revenue,
    SUM(revenue) OVER (
        PARTITION BY user_id
        ORDER BY event_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue,
    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY revenue DESC
    ) AS rank
FROM `analytics.user_revenue`;

-- CTEs (Common Table Expressions)
WITH daily_metrics AS (
    SELECT
        DATE(timestamp) AS day,
        COUNT(DISTINCT user_id) AS dau,
        COUNT(*) AS total_events
    FROM `analytics.events`
    WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    GROUP BY day
)
SELECT
    day,
    dau,
    total_events,
    AVG(dau) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS dau_7d_avg
FROM daily_metrics
ORDER BY day;

-- Approximate aggregation (faster for large datasets)
SELECT
    APPROX_COUNT_DISTINCT(user_id) AS approx_unique_users,
    APPROX_QUANTILES(latency_ms, 100)[OFFSET(99)] AS p99_latency
FROM `monitoring.requests`
WHERE date = CURRENT_DATE();
```

### Query Settings

```bash
# Dry run — estimate bytes scanned (no cost)
bq query --dry_run \
    'SELECT * FROM `analytics.events` WHERE date = "2024-01-15"'

# Use legacy SQL (not recommended)
bq query --use_legacy_sql=false \
    'SELECT COUNT(*) FROM `analytics.events`'

# Save results to destination table
bq query --destination_table=my_dataset.results \
    --replace \
    'SELECT * FROM `analytics.events` WHERE date = "2024-01-15"'

# Query with parameters (prevent SQL injection)
bq query --parameter='start_date:DATE:2024-01-01' \
    --parameter='end_date:DATE:2024-01-31' \
    'SELECT * FROM `analytics.events`
     WHERE event_date BETWEEN @start_date AND @end_date'
```

---

## Part 5 — Partitioning

### Partition Types

```
┌────────────────────────────────────────────────────────────────────┐
│         PARTITIONING                                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Partitioning divides a table into segments for faster queries:   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Table: events (partitioned by event_date)                │     │
│  │                                                            │     │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐           │     │
│  │  │ 2024-01-01 │ │ 2024-01-02 │ │ 2024-01-03 │  ...     │     │
│  │  │ 500 MB     │ │ 480 MB     │ │ 520 MB     │           │     │
│  │  └────────────┘ └────────────┘ └────────────┘           │     │
│  │                                                            │     │
│  │  Query: WHERE event_date = '2024-01-02'                  │     │
│  │  → Scans ONLY 480 MB (not entire table)                  │     │
│  │  → Huge cost and performance savings                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Partition types:                                                   │
│  1. Time-unit (HOUR, DAY, MONTH, YEAR)                            │
│     → on TIMESTAMP, DATE, or DATETIME column                      │
│  2. Ingestion time (_PARTITIONTIME pseudo-column)                 │
│  3. Integer range (on INT64 column)                                │
│                                                                      │
│  Limits:                                                            │
│  • Max 4,000 partitions per table                                 │
│  • Max 10,000 partition modifications per table per day            │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```sql
-- Create time-partitioned table
CREATE TABLE `my_dataset.events`
(
    event_id STRING,
    user_id STRING,
    event_type STRING,
    event_date DATE,
    payload JSON,
    created_at TIMESTAMP
)
PARTITION BY event_date
OPTIONS(
    partition_expiration_days = 365,   -- auto-delete after 1 year
    require_partition_filter = true     -- force WHERE on partition
);

-- Create ingestion-time partitioned table
CREATE TABLE `my_dataset.logs`
(
    message STRING,
    severity STRING
)
PARTITION BY _PARTITIONDATE;

-- Create integer-range partitioned table
CREATE TABLE `my_dataset.customers`
(
    customer_id INT64,
    name STRING,
    region STRING
)
PARTITION BY RANGE_BUCKET(customer_id, GENERATE_ARRAY(0, 1000000, 10000));

-- Query with partition filter (eliminates unnecessary scans)
SELECT * FROM `my_dataset.events`
WHERE event_date = '2024-01-15';       -- scans 1 partition only

-- Check partition info
SELECT
    table_name,
    partition_id,
    total_rows,
    total_logical_bytes
FROM `my_dataset.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events';
```

---

## Part 6 — Clustering

### Clustered Tables

```
┌────────────────────────────────────────────────────────────────────┐
│         CLUSTERING                                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Clustering sorts data within partitions for even faster scans:  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Partition: 2024-01-15                                     │     │
│  │                                                            │     │
│  │  Without clustering:          With clustering by region:  │     │
│  │  ┌──────────────────┐        ┌──────────────────┐        │     │
│  │  │ US, EU, AP, US   │        │ AP, AP, AP       │ ← skip│     │
│  │  │ AP, EU, US, AP   │        │ EU, EU, EU       │ ← skip│     │
│  │  │ EU, US, AP, EU   │        │ US, US, US, US   │ ← scan│     │
│  │  └──────────────────┘        └──────────────────┘        │     │
│  │  Scans everything             Scans only US blocks       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Up to 4 clustering columns per table.                             │
│  Order matters: cluster by most-filtered column first.            │
│  Clustering is automatically maintained by BigQuery.              │
│                                                                      │
│  Best for:                                                          │
│  • Columns frequently used in WHERE / JOIN / GROUP BY             │
│  • Columns with high cardinality                                  │
│  • Combined with partitioning for maximum savings                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```sql
-- Create partitioned + clustered table
CREATE TABLE `my_dataset.events`
(
    event_id STRING,
    user_id STRING,
    event_type STRING,
    region STRING,
    event_date DATE,
    created_at TIMESTAMP
)
PARTITION BY event_date
CLUSTER BY region, event_type;         -- up to 4 columns

-- This query benefits from BOTH partition pruning AND cluster pruning:
SELECT COUNT(*) AS events
FROM `my_dataset.events`
WHERE event_date = '2024-01-15'        -- partition pruning
  AND region = 'us-east1'              -- cluster pruning
  AND event_type = 'purchase';         -- cluster pruning (2nd level)
```

### Partitioning vs Clustering

| Aspect | Partitioning | Clustering |
|--------|-------------|------------|
| Column types | DATE, TIMESTAMP, DATETIME, INT64 | Any type |
| Max columns | 1 | 4 |
| Cost estimate | Exact (before query) | Approximate |
| Pruning | Strict boundary | Block-level min/max |
| Use together | Yes — partition first, then cluster | Yes |
| Maintenance | Automatic | Automatic (re-clustering) |

---

## Part 7 — Views & Materialized Views

### Standard Views

```sql
-- Create a view (logical, no storage cost)
CREATE VIEW `my_dataset.active_users` AS
SELECT
    user_id,
    MAX(last_login) AS last_active,
    COUNT(*) AS total_events
FROM `my_dataset.events`
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY user_id
HAVING COUNT(*) > 10;

-- Query the view (executes underlying SQL each time)
SELECT * FROM `my_dataset.active_users`
WHERE last_active >= '2024-01-01';
```

### Materialized Views

```sql
-- Create a materialized view (cached, auto-refreshed)
CREATE MATERIALIZED VIEW `my_dataset.daily_revenue`
PARTITION BY date
CLUSTER BY region
AS
SELECT
    DATE(created_at) AS date,
    region,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM `my_dataset.orders`
GROUP BY date, region;

-- BigQuery auto-uses materialized view even when querying base table:
-- This query is automatically rewritten to use the MV
SELECT region, SUM(amount) FROM `my_dataset.orders`
WHERE DATE(created_at) = '2024-01-15'
GROUP BY region;
-- ↑ BigQuery optimizer: "I can answer this from the MV instead!"
```

| Feature | View | Materialized View |
|---------|------|-------------------|
| Storage cost | None | Yes (stores results) |
| Query speed | Same as base query | Much faster (pre-computed) |
| Auto-refresh | N/A | Yes (within 5 min of base table change) |
| Smart tuning | No | Yes (auto-rewrite queries to use MV) |
| Limitations | None | Restricted SQL (no JOINs in some cases) |

---

## Part 8 — BI Engine

### In-Memory Acceleration

```
┌────────────────────────────────────────────────────────────────────┐
│         BI ENGINE                                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  BI Engine is an in-memory analysis service:                       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Looker / Data Studio / Connected Sheets                  │     │
│  │       │ SQL query                                         │     │
│  │       ▼                                                    │     │
│  │  ┌──────────────────┐                                    │     │
│  │  │ BI Engine Cache  │  ← in-memory, sub-second          │     │
│  │  │ (hot data)       │                                    │     │
│  │  └────────┬─────────┘                                    │     │
│  │           │ cache miss                                    │     │
│  │           ▼                                               │     │
│  │  ┌──────────────────┐                                    │     │
│  │  │ BigQuery Storage │  ← full scan                      │     │
│  │  └──────────────────┘                                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Reservation: allocate memory (1 GiB–100 GiB)                    │
│  Pricing: $0.0416/GiB/hour (~$30/GiB/month)                      │
│  Automatic: BQ auto-decides what to cache                         │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create BI Engine reservation
bq update --project_id=my-project \
    --bi_engine_reservation \
    --reservation_size=10  # 10 GiB
```

---

## Part 9 — Streaming Inserts & Storage Write API

### Real-Time Data Ingestion

```python
# Streaming inserts (legacy — simpler but more expensive)
from google.cloud import bigquery

client = bigquery.Client()
table_id = "my-project.analytics.events"

rows = [
    {"user_id": "u123", "event_type": "click", "timestamp": "2024-01-15T10:30:00Z"},
    {"user_id": "u456", "event_type": "purchase", "timestamp": "2024-01-15T10:31:00Z"},
]

errors = client.insert_rows_json(table_id, rows)
if errors:
    print(f"Errors: {errors}")
```

```python
# Storage Write API (recommended — cheaper, exactly-once)
from google.cloud.bigquery_storage_v1 import BigQueryWriteClient
from google.cloud.bigquery_storage_v1 import types
from google.protobuf import descriptor_pb2

client = BigQueryWriteClient()
parent = f"projects/my-project/datasets/analytics/tables/events"

# Create write stream (PENDING for exactly-once, COMMITTED for at-least-once)
write_stream = types.WriteStream(type_=types.WriteStream.Type.COMMITTED)
write_stream = client.create_write_stream(
    parent=parent, write_stream=write_stream
)

# Append rows to stream...
```

| Feature | Streaming Inserts | Storage Write API |
|---------|------------------|-------------------|
| Pricing | $0.05/GiB | $0.025/GiB (50% cheaper) |
| Delivery | At-least-once | Exactly-once (PENDING mode) |
| Availability | Immediate | Immediate (COMMITTED mode) |
| Throughput | Lower | Higher (batched) |
| Free tier | None | 2 TiB/month free |

---

## Part 10 — External Tables & Federated Queries

### Querying External Data

```sql
-- Create external table over GCS (Parquet)
CREATE EXTERNAL TABLE `my_dataset.external_logs`
OPTIONS (
    format = 'PARQUET',
    uris = ['gs://my-bucket/logs/year=*/month=*/*.parquet'],
    hive_partition_uri_prefix = 'gs://my-bucket/logs/'
);

-- Query Cloud SQL from BigQuery (federated)
SELECT * FROM
EXTERNAL_QUERY(
    'projects/my-project/locations/us/connections/my-cloudsql',
    'SELECT id, name, email FROM users WHERE active = true'
);

-- Create BigLake table (managed external with fine-grained access)
CREATE EXTERNAL TABLE `my_dataset.biglake_data`
WITH CONNECTION `projects/my-project/locations/us/connections/biglake-conn`
OPTIONS (
    format = 'PARQUET',
    uris = ['gs://data-lake-bucket/curated/*.parquet']
);
```

---

## Part 11 — BigQuery ML (BQML)

### Machine Learning in SQL

```sql
-- Create a logistic regression model
CREATE OR REPLACE MODEL `my_dataset.churn_model`
OPTIONS(
    model_type = 'LOGISTIC_REG',
    input_label_cols = ['churned'],
    auto_class_weights = TRUE,
    max_iterations = 20
) AS
SELECT
    tenure_months,
    monthly_charges,
    total_charges,
    contract_type,
    payment_method,
    churned
FROM `my_dataset.customers`
WHERE signup_date < '2024-01-01';

-- Evaluate the model
SELECT * FROM ML.EVALUATE(MODEL `my_dataset.churn_model`);

-- Make predictions
SELECT
    customer_id,
    predicted_churned,
    predicted_churned_probs[OFFSET(0)].prob AS churn_probability
FROM ML.PREDICT(
    MODEL `my_dataset.churn_model`,
    (SELECT * FROM `my_dataset.customers`
     WHERE signup_date >= '2024-01-01')
);

-- Supported model types:
-- LINEAR_REG, LOGISTIC_REG, KMEANS, BOOSTED_TREE_CLASSIFIER,
-- BOOSTED_TREE_REGRESSOR, DNN_CLASSIFIER, DNN_REGRESSOR,
-- AUTOML_CLASSIFIER, AUTOML_REGRESSOR, ARIMA_PLUS (forecasting),
-- MATRIX_FACTORIZATION, PCA, TRANSFORM, TENSORFLOW (import)
```

---

## Part 12 — Access Control & Security

### IAM & Column/Row-Level Security

```
┌────────────────────────────────────────────────────────────────────┐
│         ACCESS CONTROL                                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Levels of access control:                                          │
│                                                                      │
│  1. PROJECT level                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ roles/bigquery.admin         │ Full control              │     │
│  │ roles/bigquery.dataEditor    │ Read + write data         │     │
│  │ roles/bigquery.dataViewer    │ Read data only            │     │
│  │ roles/bigquery.jobUser       │ Run queries               │     │
│  │ roles/bigquery.user          │ Run queries + list datasets│     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  2. DATASET level                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ Grant access to specific datasets per user/group         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  3. TABLE / VIEW level                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ Grant access to specific tables using authorized views   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  4. COLUMN level (policy tags)                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ Tag sensitive columns → restrict who can see them        │     │
│  │ Example: mask email, SSN columns                         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  5. ROW level (row access policies)                                │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ Filter rows based on user identity                       │     │
│  │ Example: user sees only their region's data              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```sql
-- Row-level security
CREATE ROW ACCESS POLICY region_filter
ON `my_dataset.sales`
GRANT TO ('group:us-team@company.com')
FILTER USING (region = 'US');

CREATE ROW ACCESS POLICY eu_filter
ON `my_dataset.sales`
GRANT TO ('group:eu-team@company.com')
FILTER USING (region = 'EU');

-- Column-level security (via Data Catalog policy tags)
-- 1. Create a taxonomy in Data Catalog
-- 2. Create policy tags (e.g., "PII", "Sensitive")
-- 3. Assign policy tags to columns
-- 4. Grant "Fine-Grained Reader" role to authorized users
```

---

## Part 13 — Cost Optimization

### Cost Strategies

```
┌────────────────────────────────────────────────────────────────────┐
│         COST OPTIMIZATION                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. REDUCE BYTES SCANNED                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • SELECT only needed columns (never SELECT *)             │     │
│  │ • Use partitioned tables + partition filters             │     │
│  │ • Use clustered tables for frequent filter columns       │     │
│  │ • Use materialized views for repeated aggregations       │     │
│  │ • Preview data with LIMIT (still scans full table!)     │     │
│  │   → Use _PARTITIONTIME or table preview instead          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  2. CONTROL COSTS                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Set maximum bytes billed per query                     │     │
│  │ • Use dry runs to estimate cost before executing         │     │
│  │ • Set custom cost controls per project/user              │     │
│  │ • Monitor with INFORMATION_SCHEMA.JOBS                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  3. OPTIMIZE STORAGE                                               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Set partition expiration (auto-delete old data)        │     │
│  │ • Set table expiration for temp tables                   │     │
│  │ • Long-term storage: 50% cheaper after 90 days          │     │
│  │ • Use table clones (zero-cost copy-on-write)            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  4. CHOOSE RIGHT PRICING MODEL                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ On-demand: < 1 TiB/month scanned → use free tier        │     │
│  │ Editions:  > 1 TiB/month → consider slot reservations   │     │
│  │ Flat-rate gives predictable costs for heavy workloads   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```sql
-- Set maximum bytes billed (fail if exceeds limit)
-- In query settings or:
ALTER PROJECT SET OPTIONS(
    `region-us.default_query_max_bytes_billed` = 1099511627776  -- 1 TiB
);

-- Monitor query costs
SELECT
    user_email,
    job_id,
    query,
    total_bytes_billed,
    ROUND(total_bytes_billed / POW(1024, 4), 2) AS tib_billed,
    ROUND(total_bytes_billed / POW(1024, 4) * 6.25, 2) AS estimated_cost_usd
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY total_bytes_billed DESC
LIMIT 20;
```

---

## Part 14 — Terraform & bq CLI Reference

### Terraform

```hcl
# ─── Dataset ──────────────────────────────────────────────────
resource "google_bigquery_dataset" "analytics" {
  dataset_id    = "analytics"
  project       = var.project_id
  location      = "US"
  friendly_name = "Analytics Dataset"
  description   = "Central analytics dataset"

  default_table_expiration_ms    = null
  default_partition_expiration_ms = 31536000000  # 365 days

  labels = {
    environment = "production"
    team        = "data"
  }

  access {
    role          = "OWNER"
    special_group = "projectOwners"
  }
  access {
    role           = "READER"
    group_by_email = "data-analysts@company.com"
  }
}

# ─── Partitioned + Clustered Table ────────────────────────────
resource "google_bigquery_table" "events" {
  dataset_id = google_bigquery_dataset.analytics.dataset_id
  table_id   = "events"
  project    = var.project_id

  time_partitioning {
    type                     = "DAY"
    field                    = "event_date"
    expiration_ms            = 31536000000  # 365 days
    require_partition_filter = true
  }

  clustering = ["region", "event_type"]

  schema = jsonencode([
    { name = "event_id",   type = "STRING",    mode = "REQUIRED" },
    { name = "user_id",    type = "STRING",    mode = "REQUIRED" },
    { name = "event_type", type = "STRING",    mode = "REQUIRED" },
    { name = "region",     type = "STRING",    mode = "NULLABLE" },
    { name = "event_date", type = "DATE",      mode = "REQUIRED" },
    { name = "payload",    type = "JSON",      mode = "NULLABLE" },
    { name = "created_at", type = "TIMESTAMP", mode = "REQUIRED" },
  ])

  deletion_protection = true
}

# ─── Materialized View ───────────────────────────────────────
resource "google_bigquery_table" "daily_summary" {
  dataset_id = google_bigquery_dataset.analytics.dataset_id
  table_id   = "daily_summary_mv"
  project    = var.project_id

  materialized_view {
    query = <<-SQL
      SELECT
        event_date,
        region,
        event_type,
        COUNT(*) AS event_count,
        COUNT(DISTINCT user_id) AS unique_users
      FROM `${var.project_id}.analytics.events`
      GROUP BY event_date, region, event_type
    SQL
    enable_refresh     = true
    refresh_interval_ms = 300000  # 5 minutes
  }
}

# ─── Scheduled Query ─────────────────────────────────────────
resource "google_bigquery_data_transfer_config" "daily_report" {
  display_name   = "Daily Revenue Report"
  project        = var.project_id
  location       = "US"
  data_source_id = "scheduled_query"
  schedule       = "every day 06:00"

  destination_dataset_id = google_bigquery_dataset.analytics.dataset_id

  params = {
    query                 = "SELECT DATE(created_at) AS date, SUM(amount) AS revenue FROM `sales.orders` WHERE DATE(created_at) = @run_date GROUP BY date"
    destination_table_name_template = "daily_revenue_{run_date}"
    write_disposition               = "WRITE_TRUNCATE"
  }
}
```

### bq CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# DATASETS
# ═══════════════════════════════════════════════════════════════
bq mk --dataset --location=US my_project:my_dataset
bq ls
bq show my_dataset
bq rm -r -f my_dataset                # -r = recursive, -f = force

# ═══════════════════════════════════════════════════════════════
# TABLES
# ═══════════════════════════════════════════════════════════════
bq mk --table my_dataset.my_table schema.json
bq show --schema my_dataset.my_table
bq head -n 10 my_dataset.my_table     # preview rows
bq rm -f my_dataset.my_table

# ═══════════════════════════════════════════════════════════════
# LOADING
# ═══════════════════════════════════════════════════════════════
bq load --source_format=FORMAT DATASET.TABLE SOURCE [SCHEMA]
bq extract --destination_format=FORMAT DATASET.TABLE gs://bucket/path

# ═══════════════════════════════════════════════════════════════
# QUERYING
# ═══════════════════════════════════════════════════════════════
bq query 'SELECT ...'
bq query --dry_run 'SELECT ...'
bq query --destination_table=DATASET.TABLE 'SELECT ...'
bq query --max_bytes_billed=1000000000 'SELECT ...'

# ═══════════════════════════════════════════════════════════════
# JOBS
# ═══════════════════════════════════════════════════════════════
bq ls -j --max_results=10
bq show -j JOB_ID
bq cancel JOB_ID

# ═══════════════════════════════════════════════════════════════
# COPY
# ═══════════════════════════════════════════════════════════════
bq cp source_dataset.table dest_dataset.table
bq cp --clone source_dataset.table dest_dataset.clone_table
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Event Analytics Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: EVENT ANALYTICS                                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────────┐       │
│  │ Mobile   │  │ Web App  │  │ Backend   │  │ IoT Devices  │       │
│  │ App      │  │          │  │ Services  │  │              │       │
│  └────┬─────┘  └────┬─────┘  └─────┬─────┘  └──────┬───────┘       │
│       └──────────────┼──────────────┼───────────────┘               │
│                      ▼                                                │
│              ┌───────────────┐                                       │
│              │ Pub/Sub       │                                       │
│              │ (event stream)│                                       │
│              └───────┬───────┘                                       │
│                      ▼                                                │
│              ┌───────────────┐                                       │
│              │ Dataflow      │  streaming pipeline                   │
│              │ (transform,   │  → enrich, validate, flatten         │
│              │  enrich)      │                                       │
│              └───────┬───────┘                                       │
│                      ▼                                                │
│              ┌───────────────┐                                       │
│              │ BigQuery      │  partitioned by day                   │
│              │ events table  │  clustered by event_type, user_id    │
│              └───────┬───────┘                                       │
│                      │                                                │
│           ┌──────────┼──────────┐                                    │
│           ▼          ▼          ▼                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                            │
│  │ Looker   │ │ Scheduled│ │ BQML     │                            │
│  │ Dashboard│ │ Queries  │ │ Churn    │                            │
│  │          │ │ (daily)  │ │ Model    │                            │
│  └──────────┘ └──────────┘ └──────────┘                            │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Data Lakehouse

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: DATA LAKEHOUSE                                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  DATA LAKE (Cloud Storage)                                 │        │
│  │  gs://data-lake/                                          │        │
│  │  ├── raw/          ← land raw data (any format)          │        │
│  │  ├── curated/      ← cleaned Parquet (BigLake tables)    │        │
│  │  └── archive/      ← Coldline/Archive class              │        │
│  └──────────────────────────┬───────────────────────────────┘        │
│                              │                                        │
│  ┌──────────────────────────▼───────────────────────────────┐        │
│  │  BigQuery                                                  │        │
│  │                                                            │        │
│  │  BigLake tables → query Parquet in GCS with BQ SQL       │        │
│  │  Native tables  → high-performance analytics             │        │
│  │  External tables → federated queries to Cloud SQL        │        │
│  │                                                            │        │
│  │  Row/column security → fine-grained access control       │        │
│  │  Materialized views → pre-computed aggregations          │        │
│  │  BI Engine → sub-second dashboards                       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Benefits:                                                            │
│  • Open format (Parquet) — no vendor lock-in                        │
│  • BigLake gives BigQuery-level security on GCS data                │
│  • Cost: store in GCS (cheap), query with BQ (pay per query)       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Multi-Region Analytics

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: MULTI-REGION WITH COST CONTROL                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Dataset: analytics (location: US)                         │        │
│  │                                                            │        │
│  │  events (partitioned by day, clustered by region)        │        │
│  │  ├── require_partition_filter = true                     │        │
│  │  ├── partition_expiration = 365 days                     │        │
│  │  └── long-term storage auto-applies after 90 days       │        │
│  │                                                            │        │
│  │  daily_summary_mv (materialized view)                    │        │
│  │  └── dashboards query this instead of base table         │        │
│  │                                                            │        │
│  │  Cost controls:                                           │        │
│  │  ├── max_bytes_billed = 10 TiB per query                │        │
│  │  ├── Custom quotas per user/project                      │        │
│  │  ├── Scheduled queries for batch reports                 │        │
│  │  └── BI Engine for interactive dashboards (cached)       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Cross-region authorized views:                                      │
│  • EU dataset has authorized views into US dataset                  │
│  • EU team sees only EU data (row-level security)                   │
│  • Avoids duplicating data across regions                            │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command / SQL |
|--------|-------------|
| Create dataset | `bq mk --dataset --location=US project:dataset` |
| Load CSV | `bq load --source_format=CSV --autodetect dataset.table gs://bucket/file.csv` |
| Load Parquet | `bq load --source_format=PARQUET dataset.table gs://bucket/*.parquet` |
| Dry run | `bq query --dry_run 'SELECT ...'` |
| Partition by day | `PARTITION BY event_date` |
| Cluster | `CLUSTER BY col1, col2` |
| Create MV | `CREATE MATERIALIZED VIEW ... AS SELECT ...` |
| Row security | `CREATE ROW ACCESS POLICY ... FILTER USING (condition)` |
| Export | `bq extract dataset.table gs://bucket/output.csv` |
| Copy table | `bq cp src.table dest.table` |
| Clone table | `bq cp --clone src.table dest.clone` |
| Max bytes billed | `--max_bytes_billed=N` |
| Free tier | 1 TiB queries + 10 GiB storage/month |

---

## What is BigQuery? (Beginner Explanation)

### Simple Analogy

BigQuery is like a **giant spreadsheet in the cloud** that can hold **billions of rows** and let you ask questions (queries) in **seconds**. Imagine Google Sheets, but instead of lagging at 100,000 rows, it handles petabytes of data without breaking a sweat.

You don't install anything, don't manage servers, and don't worry about storage space. You just upload your data, write SQL questions, and get answers — Google handles everything behind the scenes.

### Why BigQuery is Special

```
┌────────────────────────────────────────────────────────────────────┐
│         WHY BIGQUERY IS SPECIAL                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. SERVERLESS                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ No servers to provision, manage, or scale                 │     │
│  │ No indexes to create or tune                             │     │
│  │ No vacuuming, no maintenance windows                     │     │
│  │ Just write SQL → get results                             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  2. NO INFRASTRUCTURE                                               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ You never see a VM, disk, or cluster                      │     │
│  │ Google auto-scales thousands of CPUs for your query      │     │
│  │ Storage and compute scale independently                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  3. JUST WRITE SQL                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ Standard SQL — same syntax you already know              │     │
│  │ No proprietary language to learn                          │     │
│  │ Built-in ML, geospatial, and JSON support                │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  4. PAY FOR WHAT YOU USE                                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ On-demand: pay only for data scanned by your query       │     │
│  │ No charge when you're not querying                        │     │
│  │ 1 TiB free per month                                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### When to Use BigQuery vs Cloud SQL

| Aspect | BigQuery | Cloud SQL |
|--------|----------|-----------|
| **Purpose** | Analytics, reporting, data warehousing | Transactional apps (CRUD operations) |
| **Data size** | Gigabytes to petabytes | Megabytes to a few terabytes |
| **Query type** | Complex aggregations over huge datasets | Simple lookups, inserts, updates |
| **Latency** | Seconds (batch-oriented) | Milliseconds (real-time) |
| **Use case** | "How many users signed up last quarter?" | "Get user profile by ID" |
| **Pricing** | Pay per query (data scanned) | Pay per instance (always running) |
| **Indexes** | Not needed (columnar scan) | Required for performance |
| **Joins** | Fast on billions of rows | Slows down on large datasets |
| **Example** | Dashboard, reports, ML training data | Web app backend, user auth, orders |

**Rule of thumb:**
- Need to **analyze** large data? → **BigQuery**
- Need to **serve** an application with fast reads/writes? → **Cloud SQL**
- Many teams use **both**: Cloud SQL for the app, BigQuery for analytics (sync data via scheduled exports or Dataflow)

---

## Console Walkthrough: Using BigQuery

### Step 1 — Opening BigQuery Console

```
Console → BigQuery
   OR
Console → Navigation menu (☰) → Analytics → BigQuery
```

You land on the **BigQuery Studio** page with:
- **Explorer panel** (left) — lists your projects, datasets, and tables
- **Query Editor** (center) — where you write and run SQL
- **Results panel** (bottom) — shows query results and execution details

### Step 2 — Creating a Dataset

```
Explorer panel → Click the three dots (⋮) next to your project
   → "Create dataset"
```

Fill in the fields:

| Field | What to Enter | Example |
|-------|--------------|---------|
| **Dataset ID** | Name for your dataset (letters, numbers, underscores) | `sales_data` |
| **Location type** | Region or Multi-region | Multi-region |
| **Data location** | Where to store data | `US` |
| **Default table expiration** | Auto-delete tables after N days (optional) | `Never` or `90 days` |
| **Default max time travel** | How far back you can query historical data | `7 days` (default) |
| **Encryption** | Google-managed key or CMEK | Google-managed (default) |

Click **"Create dataset"** → the dataset appears under your project in the Explorer panel.

### Step 3 — Creating a Table

```
Explorer panel → Click the three dots (⋮) next to your dataset
   → "Create table"
```

**Option A: Upload a CSV file**

| Field | What to Enter |
|-------|--------------|
| **Create table from** | Upload |
| **Select file** | Browse and select your CSV file |
| **File format** | CSV |
| **Table name** | `my_table` |
| **Schema** | Check "Auto detect" or define columns manually |

**Option B: From Google Sheets**

| Field | What to Enter |
|-------|--------------|
| **Create table from** | Google Drive |
| **Select Drive URI** | Paste the Google Sheets URL |
| **File format** | Google Sheets |
| **Table name** | `sheets_data` |
| **Sheet range** | (optional) Specific sheet or cell range |

**Option C: Empty table (define schema manually)**

| Field | What to Enter |
|-------|--------------|
| **Create table from** | Empty table |
| **Table name** | `events` |
| **Schema** | Click "+ Add field" for each column |

For each column, specify:
- **Name** — column name (e.g., `user_id`)
- **Type** — STRING, INT64, FLOAT64, DATE, TIMESTAMP, etc.
- **Mode** — NULLABLE, REQUIRED, or REPEATED

Optional settings under **Advanced options**:
- **Partitioning** — select a DATE/TIMESTAMP column
- **Clustering** — select up to 4 columns
- **Table expiration** — auto-delete date

Click **"Create table"**.

### Step 4 — Running a Query in the Query Editor

```
Click "+ Compose a new query" (or use the editor)
```

Type your SQL:

```sql
SELECT
    product_name,
    SUM(quantity) AS total_sold,
    SUM(price * quantity) AS total_revenue
FROM `my-project.sales_data.orders`
GROUP BY product_name
ORDER BY total_revenue DESC
LIMIT 10;
```

Click **"Run"** (or press `Ctrl+Enter`).

**Before running**, BigQuery shows a green validator message:
- ✅ "This query will process **X MB**" — estimated data scanned
- ❌ Red error message if SQL has syntax errors

### Step 5 — Viewing Query Results and Execution Details

After the query runs, you see two tabs:

**Results tab:**
- Table of query results
- Download as CSV or JSON
- "Save results" → save to BigQuery table, Google Sheets, or local file

**Execution details tab (click "Execution details"):**

| Detail | What It Shows |
|--------|--------------|
| **Bytes processed** | How much data was scanned (you pay for this) |
| **Bytes billed** | Rounded up to nearest 10 MB (minimum 10 MB) |
| **Estimated cost** | Dollar amount based on on-demand pricing |
| **Slot time** | Total compute time consumed |
| **Elapsed time** | Wall-clock time for the query |
| **Stages** | Execution plan with read/compute/write stages |

### Step 6 — Saving and Scheduling Queries

**Saving a query:**
```
Query Editor → "Save" → "Save query"
   → Enter a name → Save
```
Saved queries appear under **"Saved queries"** in the Explorer panel.

**Scheduling a query:**
```
Query Editor → "Schedule" → "Create new scheduled query"
```

| Field | What to Enter |
|-------|--------------|
| **Name** | `Daily Sales Report` |
| **Schedule** | Repeats: Daily, at 6:00 AM |
| **Destination table** | Dataset and table name for results |
| **Write preference** | Append or Overwrite |

Click **"Save"** → the query runs automatically on schedule.

### Step 7 — Deleting Tables and Datasets

**Delete a table:**
```
Explorer panel → Click on the table
   → Click "Delete" (🗑️ icon) in the table details
   → Type the table name to confirm → Delete
```

**Delete a dataset:**
```
Explorer panel → Click the three dots (⋮) next to the dataset
   → "Delete dataset"
   → Type the dataset ID to confirm → Delete
```

> ⚠️ **Warning**: Deleting a dataset with **"Delete table contents"** checked will delete ALL tables, views, and data inside it. This cannot be undone (unless you have time travel within the window).

---

## Additional bq Commands

### bq update — Modify Table Schema and Dataset Properties

```bash
# Add a new column to an existing table
bq update my_dataset.my_table \
    schema_with_new_column.json

# Update dataset description
bq update --description="Updated analytics dataset" \
    my_dataset

# Update dataset default expiration (in seconds)
bq update --default_table_expiration=7776000 \
    my_dataset                                 # 90 days

# Set partition expiration on a table (in seconds)
bq update --time_partitioning_expiration=31536000 \
    my_dataset.events                          # 365 days

# Add a label to a dataset
bq update --set_label=team:analytics \
    my_dataset

# Add a label to a table
bq update --set_label=env:production \
    my_dataset.events
```

> **Note**: You can add new columns to a table schema, but you cannot remove or rename existing columns. To change a column, create a new table with the updated schema and copy the data.

### bq rm — Delete Tables, Datasets, and Views

```bash
# Delete a table (prompts for confirmation)
bq rm my_dataset.my_table

# Delete a table without confirmation
bq rm -f my_dataset.my_table

# Delete a view
bq rm -f my_dataset.my_view

# Delete an entire dataset (must be empty)
bq rm -d my_dataset

# Delete a dataset and ALL its contents (tables, views, models)
bq rm -r -f my_dataset

# Delete a specific partition
bq rm -f 'my_dataset.events$20240115'      # partition 2024-01-15

# Delete a model
bq rm -f --model my_dataset.churn_model
```

### bq query — Run Queries from CLI

```bash
# Run a simple query
bq query 'SELECT COUNT(*) AS total FROM `my_dataset.events`'

# Use standard SQL (recommended)
bq query --use_legacy_sql=false \
    'SELECT user_id, COUNT(*) AS cnt
     FROM `my_dataset.events`
     GROUP BY user_id
     ORDER BY cnt DESC
     LIMIT 10'

# Dry run — estimate cost without running
bq query --dry_run \
    'SELECT * FROM `my_dataset.events`
     WHERE event_date = "2024-01-15"'

# Set maximum bytes billed (fail if query exceeds limit)
bq query --max_bytes_billed=1073741824 \
    'SELECT * FROM `my_dataset.events`'    # 1 GiB limit

# Save query results to a destination table
bq query --destination_table=my_dataset.results \
    --replace \
    'SELECT * FROM `my_dataset.events`
     WHERE event_date = "2024-01-15"'

# Run query with parameters
bq query --parameter='target_date:DATE:2024-01-15' \
    'SELECT * FROM `my_dataset.events`
     WHERE event_date = @target_date'

# Format output as JSON
bq query --format=json \
    'SELECT * FROM `my_dataset.events` LIMIT 5'
```

### bq load — Load Data from File or GCS

```bash
# Load a local CSV file
bq load --source_format=CSV \
    --autodetect \
    --skip_leading_rows=1 \
    my_dataset.my_table \
    ./local_data.csv

# Load JSON from GCS
bq load --source_format=NEWLINE_DELIMITED_JSON \
    --autodetect \
    my_dataset.events \
    gs://my-bucket/events/*.json

# Load Parquet from GCS with partitioning and clustering
bq load --source_format=PARQUET \
    --time_partitioning_field=event_date \
    --clustering_fields=region,event_type \
    my_dataset.events \
    gs://my-bucket/parquet/events/

# Load with explicit schema
bq load --source_format=CSV \
    --skip_leading_rows=1 \
    my_dataset.products \
    gs://my-bucket/products.csv \
    id:INTEGER,name:STRING,price:FLOAT,in_stock:BOOLEAN

# Load and replace (overwrite) existing data
bq load --source_format=CSV \
    --autodetect \
    --replace \
    my_dataset.my_table \
    gs://my-bucket/full_export.csv

# Load with max bad records (skip errors)
bq load --source_format=CSV \
    --autodetect \
    --max_bad_records=100 \
    my_dataset.my_table \
    gs://my-bucket/messy_data.csv
```

### bq extract — Export Data to GCS

```bash
# Export table to CSV in GCS
bq extract my_dataset.my_table \
    gs://my-bucket/exports/my_table.csv

# Export to newline-delimited JSON
bq extract --destination_format=NEWLINE_DELIMITED_JSON \
    my_dataset.my_table \
    gs://my-bucket/exports/my_table.json

# Export to Parquet (recommended for large datasets)
bq extract --destination_format=PARQUET \
    my_dataset.my_table \
    gs://my-bucket/exports/my_table.parquet

# Export to Avro
bq extract --destination_format=AVRO \
    my_dataset.my_table \
    gs://my-bucket/exports/my_table.avro

# Export with compression (CSV/JSON)
bq extract --compression=GZIP \
    my_dataset.my_table \
    gs://my-bucket/exports/my_table.csv.gz

# Export large table to multiple files (use wildcard)
bq extract my_dataset.large_table \
    gs://my-bucket/exports/large_table_*.csv
```

### bq Commands Quick Reference

| Command | Description |
|---------|-------------|
| `bq update dataset` | Modify dataset properties (description, labels, expiration) |
| `bq update dataset.table schema.json` | Add new columns to table schema |
| `bq rm -f dataset.table` | Delete a table without confirmation |
| `bq rm -r -f dataset` | Delete a dataset and all its contents |
| `bq query 'SQL'` | Run a SQL query from the command line |
| `bq query --dry_run 'SQL'` | Estimate bytes scanned without running |
| `bq load --source_format=FMT dataset.table SOURCE` | Load data from file or GCS |
| `bq extract dataset.table gs://bucket/path` | Export table data to GCS |
| `bq extract --destination_format=PARQUET ...` | Export as Parquet (recommended) |

---

## What's Next?

Continue to **Chapter 55: Dataflow** → `55-dataflow.md`
