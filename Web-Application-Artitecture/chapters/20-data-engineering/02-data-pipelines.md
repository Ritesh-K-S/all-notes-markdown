# Data Pipelines (ETL & ELT)

> **What you'll learn**: How data moves from where it's created (your app's database) to where it's analyzed (your data warehouse) — and the critical difference between transforming data *before* vs *after* loading it.

---

## Real-Life Analogy

Imagine you work at a **newspaper** that collects news from reporters around the world.

**ETL (Extract, Transform, Load)** is like the old-school newsroom:
1. **Extract**: Reporters call in with raw stories from the field
2. **Transform**: Editors rewrite, fact-check, and format each story *before* publishing
3. **Load**: The polished story goes into tomorrow's printed newspaper

The editors are the bottleneck — but what goes into print is always clean and ready to read.

**ELT (Extract, Load, Transform)** is like a modern online news platform:
1. **Extract**: Reporters upload raw stories, videos, tweets directly
2. **Load**: Everything gets dumped into a massive content repository *immediately*
3. **Transform**: Different teams then reorganize, tag, and format the content for different audiences (mobile app, website, newsletter)

The raw content arrives faster, and different teams transform it differently for their needs.

---

## Core Concept Explained Step-by-Step

### Step 1: Why Do We Need Data Pipelines?

In [Chapter 20.1](./01-oltp-vs-olap.md), we learned that OLTP and OLAP are separate systems. But how does data flow from one to the other?

```
Your App                    ??? (Magic?)                    Analytics
┌──────────┐                                              ┌──────────────┐
│PostgreSQL │  ──── What goes here? ────────────────────▶  │  BigQuery    │
│(OLTP)     │                                              │  (OLAP)      │
└──────────┘                                              └──────────────┘
```

**Data pipelines** are the plumbing that moves and transforms data between systems. Without them, your analysts would have no data to analyze.

### Step 2: The Three Steps — Extract, Transform, Load

Every data pipeline performs three fundamental operations:

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE THREE STEPS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ① EXTRACT                                                        │
│     Pull data from source systems                                 │
│     Sources: databases, APIs, files, event streams                │
│                                                                   │
│  ② TRANSFORM                                                      │
│     Clean, reshape, enrich, aggregate the data                    │
│     Fix nulls, join tables, compute new columns                   │
│                                                                   │
│  ③ LOAD                                                           │
│     Write the processed data into the destination                 │
│     Destination: data warehouse, data lake, search index          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: ETL vs ELT — The Order Matters

The difference is **when** you transform:

```
ETL (Traditional):
──────────────────
  Source ──▶ [EXTRACT] ──▶ [TRANSFORM on ETL Server] ──▶ [LOAD] ──▶ Warehouse
                                    │
                           Transform happens BEFORE
                           data reaches the warehouse.
                           ETL server does the heavy lifting.


ELT (Modern):
──────────────
  Source ──▶ [EXTRACT] ──▶ [LOAD raw data] ──▶ [TRANSFORM in Warehouse] ──▶ Ready
                                                        │
                                              Transform happens INSIDE
                                              the warehouse (BigQuery, Snowflake).
                                              Warehouse's compute power does the work.
```

### Step 4: Why Did the Industry Shift from ETL to ELT?

| Era | Constraint | Solution |
|-----|-----------|----------|
| 1990s-2010s | Data warehouses were expensive (licensed per TB stored) | ETL: Transform & reduce data BEFORE loading to save warehouse cost |
| 2010s-now | Cloud warehouses are cheap to store, powerful to compute (BigQuery, Snowflake) | ELT: Load everything raw, transform using warehouse's massive compute |

**Key insight**: When storage is cheap but your ETL server is the bottleneck, move the transformation into the warehouse where compute scales elastically.

---

## How It Works Internally

### ETL Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        TRADITIONAL ETL PIPELINE                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Data Sources              ETL Server                  Data Warehouse      │
│  ───────────              ──────────                  ──────────────      │
│                                                                           │
│  ┌──────────┐         ┌──────────────────┐         ┌──────────────┐      │
│  │PostgreSQL │──────▶  │                  │         │              │      │
│  └──────────┘         │  Extract:         │         │              │      │
│                       │  • Read data      │         │              │      │
│  ┌──────────┐         │  • Track changes  │         │  Redshift    │      │
│  │  MySQL    │──────▶  │                  │──────▶  │  (clean,     │      │
│  └──────────┘         │  Transform:       │         │  structured  │      │
│                       │  • Clean nulls    │         │  data)       │      │
│  ┌──────────┐         │  • Join tables    │         │              │      │
│  │  API      │──────▶  │  • Compute cols  │         │              │      │
│  └──────────┘         │  • Aggregate     │         │              │      │
│                       │                  │         └──────────────┘      │
│  ┌──────────┐         │  Load:           │                               │
│  │  CSV/S3   │──────▶  │  • Bulk insert   │                               │
│  └──────────┘         └──────────────────┘                               │
│                                                                           │
│  Tools: Apache Airflow, Informatica, Talend, SSIS                         │
└──────────────────────────────────────────────────────────────────────────┘
```

### ELT Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         MODERN ELT PIPELINE                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Data Sources       Ingestion Layer        Data Warehouse                 │
│  ───────────       ────────────────        ──────────────                 │
│                                                                           │
│  ┌──────────┐     ┌────────────────┐     ┌────────────────────────┐      │
│  │PostgreSQL │──▶  │                │     │  Raw Layer (Landing)    │      │
│  └──────────┘     │  Fivetran /     │──▶  │  • Raw, unmodified data │      │
│                   │  Airbyte /      │     │                        │      │
│  ┌──────────┐     │  Stitch         │     ├────────────────────────┤      │
│  │  MySQL    │──▶  │                │     │  Staging Layer          │      │
│  └──────────┘     │  (Extract +     │     │  • Cleaned, typed       │      │
│                   │   Load only)    │     │                        │      │
│  ┌──────────┐     │                │     ├────────────────────────┤      │
│  │  API      │──▶  │  No transform! │     │  Marts Layer            │      │
│  └──────────┘     └────────────────┘     │  • Business logic       │      │
│                                          │  • Aggregated for users  │      │
│                                          └────────────────────────┘      │
│                                                    ▲                      │
│                                                    │                      │
│                                          ┌─────────┴──────────┐          │
│                                          │  dbt (Transform)    │          │
│                                          │  SQL-based models   │          │
│                                          │  Runs IN warehouse  │          │
│                                          └────────────────────┘          │
│                                                                           │
│  Tools: Fivetran + dbt + Snowflake/BigQuery                               │
└──────────────────────────────────────────────────────────────────────────┘
```

### Batch vs Streaming Pipelines

```
┌─────────────────────────────────────────────────────────────────┐
│                  BATCH vs STREAMING                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  BATCH (Scheduled):                                               │
│  ─────────────────                                               │
│  "Every night at 2 AM, pull yesterday's data"                     │
│                                                                   │
│  Source ══▶ [Airflow DAG at 2AM] ══▶ Warehouse                    │
│                                                                   │
│  Latency: Hours (data is stale until next run)                    │
│  Good for: Reports, dashboards, ML training                       │
│                                                                   │
│  ─────────────────────────────────────────────────────────────── │
│                                                                   │
│  STREAMING (Continuous):                                          │
│  ──────────────────────                                          │
│  "Every event flows through in real-time"                         │
│                                                                   │
│  Source ──▶ [Kafka] ──▶ [Flink/Spark Streaming] ──▶ Warehouse    │
│                                                                   │
│  Latency: Seconds to minutes                                      │
│  Good for: Real-time dashboards, fraud detection, alerting        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: ETL Pipeline with Apache Airflow

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import psycopg2
import pandas as pd
from google.cloud import bigquery

# ═══════════════════════════════════════════════════════
# ETL Pipeline: Move order data from PostgreSQL to BigQuery
# Runs daily at 2 AM
# ═══════════════════════════════════════════════════════

default_args = {
    'owner': 'data-team',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'orders_etl_pipeline',
    default_args=default_args,
    schedule_interval='0 2 * * *',  # Daily at 2 AM
    start_date=datetime(2024, 1, 1),
    catchup=False,
)


def extract(**context):
    """Step 1: Extract yesterday's orders from PostgreSQL."""
    execution_date = context['ds']  # e.g., '2024-06-15'
    
    conn = psycopg2.connect("dbname=myapp host=prod-db.internal")
    query = """
        SELECT order_id, user_id, product_id, amount, status, created_at
        FROM orders
        WHERE created_at::date = %s
    """
    df = pd.read_sql(query, conn, params=[execution_date])
    conn.close()
    
    # Save to intermediate storage
    df.to_parquet(f'/tmp/orders_{execution_date}.parquet')
    return f'/tmp/orders_{execution_date}.parquet'


def transform(**context):
    """Step 2: Clean and enrich the data."""
    execution_date = context['ds']
    df = pd.read_parquet(f'/tmp/orders_{execution_date}.parquet')
    
    # Remove cancelled orders
    df = df[df['status'] != 'CANCELLED']
    
    # Add derived columns
    df['order_date'] = pd.to_datetime(df['created_at']).dt.date
    df['order_hour'] = pd.to_datetime(df['created_at']).dt.hour
    df['is_high_value'] = df['amount'] > 1000
    
    # Handle nulls
    df['amount'] = df['amount'].fillna(0)
    
    df.to_parquet(f'/tmp/orders_transformed_{execution_date}.parquet')


def load(**context):
    """Step 3: Load transformed data into BigQuery."""
    execution_date = context['ds']
    df = pd.read_parquet(f'/tmp/orders_transformed_{execution_date}.parquet')
    
    client = bigquery.Client()
    job_config = bigquery.LoadJobConfig(
        write_disposition="WRITE_APPEND",  # Append daily data
        schema_update_options=[bigquery.SchemaUpdateOption.ALLOW_FIELD_ADDITION],
    )
    
    job = client.load_table_from_dataframe(
        df, 'project.analytics.fact_orders', job_config=job_config
    )
    job.result()  # Wait for completion
    print(f"Loaded {len(df)} rows for {execution_date}")


# Define DAG tasks in order
extract_task = PythonOperator(task_id='extract', python_callable=extract, dag=dag)
transform_task = PythonOperator(task_id='transform', python_callable=transform, dag=dag)
load_task = PythonOperator(task_id='load', python_callable=load, dag=dag)

# Set dependencies: extract → transform → load
extract_task >> transform_task >> load_task
```

### Python: ELT with dbt (Transform in Warehouse)

```sql
-- dbt model: models/marts/fact_orders_enriched.sql
-- This SQL runs INSIDE the warehouse (BigQuery/Snowflake)
-- dbt handles dependencies, testing, and documentation

{{ config(materialized='incremental', unique_key='order_id') }}

WITH raw_orders AS (
    -- Raw data loaded by Fivetran (no transformation yet)
    SELECT * FROM {{ source('postgres_replica', 'orders') }}
    {% if is_incremental() %}
    WHERE _fivetran_synced > (SELECT MAX(_fivetran_synced) FROM {{ this }})
    {% endif %}
),

enriched AS (
    SELECT
        o.order_id,
        o.user_id,
        o.product_id,
        o.amount,
        o.status,
        o.created_at,
        
        -- Derived columns (transform step)
        DATE(o.created_at) AS order_date,
        EXTRACT(HOUR FROM o.created_at) AS order_hour,
        CASE WHEN o.amount > 1000 THEN TRUE ELSE FALSE END AS is_high_value,
        
        -- Join with dimension tables
        p.category AS product_category,
        p.brand AS product_brand,
        u.signup_date AS user_signup_date,
        DATE_DIFF(DATE(o.created_at), u.signup_date, DAY) AS days_since_signup
        
    FROM raw_orders o
    LEFT JOIN {{ ref('dim_products') }} p ON o.product_id = p.product_id
    LEFT JOIN {{ ref('dim_users') }} u ON o.user_id = u.user_id
    WHERE o.status != 'CANCELLED'
)

SELECT * FROM enriched
```

### Java: Simple ETL Pipeline

```java
import java.sql.*;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

public class SimpleETLPipeline {
    
    private final Connection sourceDb;  // PostgreSQL (OLTP)
    private final Connection targetDb;  // ClickHouse (OLAP)
    
    /**
     * Extract: Pull yesterday's orders from production database.
     */
    public List<OrderRecord> extract(LocalDate date) throws SQLException {
        List<OrderRecord> records = new ArrayList<>();
        
        String query = """
            SELECT order_id, user_id, product_id, amount, status, created_at
            FROM orders WHERE created_at::date = ?
        """;
        
        PreparedStatement ps = sourceDb.prepareStatement(query);
        ps.setDate(1, java.sql.Date.valueOf(date));
        ResultSet rs = ps.executeQuery();
        
        while (rs.next()) {
            records.add(new OrderRecord(
                rs.getLong("order_id"),
                rs.getLong("user_id"),
                rs.getInt("product_id"),
                rs.getBigDecimal("amount"),
                rs.getString("status"),
                rs.getTimestamp("created_at").toLocalDateTime()
            ));
        }
        System.out.printf("Extracted %d records for %s%n", records.size(), date);
        return records;
    }
    
    /**
     * Transform: Clean data, compute derived fields, filter.
     */
    public List<TransformedOrder> transform(List<OrderRecord> raw) {
        return raw.stream()
            .filter(r -> !"CANCELLED".equals(r.status()))  // Remove cancelled
            .map(r -> new TransformedOrder(
                r.orderId(),
                r.userId(),
                r.productId(),
                r.amount(),
                r.createdAt().toLocalDate(),         // Extract date
                r.createdAt().getHour(),             // Extract hour
                r.amount().compareTo(new java.math.BigDecimal("1000")) > 0  // High value flag
            ))
            .toList();
    }
    
    /**
     * Load: Bulk insert into analytical database (ClickHouse).
     */
    public void load(List<TransformedOrder> data) throws SQLException {
        String insert = """
            INSERT INTO fact_orders 
            (order_id, user_id, product_id, amount, order_date, order_hour, is_high_value)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """;
        
        PreparedStatement ps = targetDb.prepareStatement(insert);
        int batchSize = 0;
        
        for (TransformedOrder order : data) {
            ps.setLong(1, order.orderId());
            ps.setLong(2, order.userId());
            ps.setInt(3, order.productId());
            ps.setBigDecimal(4, order.amount());
            ps.setDate(5, java.sql.Date.valueOf(order.orderDate()));
            ps.setInt(6, order.orderHour());
            ps.setBoolean(7, order.isHighValue());
            ps.addBatch();
            
            if (++batchSize % 10000 == 0) {
                ps.executeBatch();  // Bulk insert every 10K rows
            }
        }
        ps.executeBatch();  // Final batch
        System.out.printf("Loaded %d records into warehouse%n", data.size());
    }
    
    /** Run the full ETL pipeline. */
    public void run(LocalDate date) throws SQLException {
        List<OrderRecord> raw = extract(date);
        List<TransformedOrder> transformed = transform(raw);
        load(transformed);
    }
}
```

---

## Infrastructure Examples

### Apache Airflow DAG Visualization

```
┌─────────────────────────────────────────────────────────────┐
│                  AIRFLOW DAG: daily_orders_pipeline          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [check_source_ready] ──▶ [extract_orders] ──┐              │
│                                              │              │
│  [check_source_ready] ──▶ [extract_users]  ──┼──▶ [transform] ──▶ [load] ──▶ [notify_slack]
│                                              │              │
│  [check_source_ready] ──▶ [extract_products]─┘              │
│                                                              │
│  Schedule: Daily @ 02:00 UTC                                 │
│  SLA: Must complete by 06:00 UTC                             │
│  Retries: 3 with exponential backoff                         │
└─────────────────────────────────────────────────────────────┘
```

### Modern Data Stack (ELT Architecture)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       THE MODERN DATA STACK                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SOURCES              INGESTION          WAREHOUSE         TRANSFORM     │
│  ───────              ─────────          ─────────         ─────────     │
│                                                                          │
│  PostgreSQL ─────┐                    ┌───────────────┐                  │
│  MySQL ──────────┼───▶ Fivetran ────▶ │  Snowflake    │ ◀── dbt         │
│  Stripe API ─────┤    (EL tool)       │  ┌─────────┐ │    (Transform)  │
│  Salesforce ─────┤                    │  │Raw Layer│ │         │        │
│  Google Ads ─────┘                    │  │Staging  │ │         ▼        │
│                                       │  │Marts    │ │   ┌──────────┐  │
│                                       │  └─────────┘ │   │  Looker  │  │
│                                       └───────────────┘   │  Metabase│  │
│                                                           │  Tableau │  │
│                                                           └──────────┘  │
│                                                            (BI Tools)    │
│                                                                          │
│  ORCHESTRATION: Airflow / Dagster / Prefect                              │
│  QUALITY: Great Expectations / dbt tests / Monte Carlo                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ETL vs ELT Comparison

| Aspect | ETL | ELT |
|--------|-----|-----|
| **Transform location** | Separate ETL server | Inside the data warehouse |
| **Data in warehouse** | Clean, structured only | Raw + transformed layers |
| **Flexibility** | Must re-engineer pipeline to change logic | Change SQL models, re-run |
| **Scalability** | Limited by ETL server capacity | Warehouse scales elastically |
| **Cost** | ETL server compute costs | Warehouse compute costs |
| **Latency** | Higher (transform before load) | Lower (load immediately) |
| **Schema changes** | Painful (rebuild pipelines) | Easy (just update SQL models) |
| **Best for** | On-premise, regulated environments | Cloud-native, agile teams |
| **Tools** | Informatica, Talend, SSIS | Fivetran + dbt, Airbyte + dbt |

---

## Real-World Example

### Spotify's Data Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SPOTIFY's DATA PIPELINE                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  User streams a song                                                  │
│         │                                                             │
│         ▼                                                             │
│  ┌─────────────────┐                                                  │
│  │  Event Logging   │  (Billions of events/day)                        │
│  │  (Kafka)         │                                                  │
│  └────────┬────────┘                                                  │
│           │                                                           │
│     ┌─────┴─────┐                                                     │
│     ▼           ▼                                                     │
│  [Real-Time]  [Batch]                                                 │
│     │           │                                                     │
│     ▼           ▼                                                     │
│  ┌────────┐  ┌──────────────┐                                         │
│  │ Flink   │  │ Spark (ETL)   │                                         │
│  │ (fraud, │  │ Daily jobs     │                                         │
│  │ live    │  │ Billions rows  │                                         │
│  │ metrics)│  └──────┬───────┘                                         │
│  └────────┘         │                                                 │
│                     ▼                                                 │
│           ┌─────────────────┐                                         │
│           │  Data Lake (GCS) │                                         │
│           │  Petabytes of    │                                         │
│           │  listening data  │                                         │
│           └────────┬────────┘                                         │
│                    │                                                   │
│                    ▼                                                   │
│           ┌─────────────────┐                                         │
│           │  BigQuery        │  ◀── Analysts run queries               │
│           │  (Warehouse)     │  ◀── ML models train here               │
│           └─────────────────┘      (Discover Weekly recommendations)   │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

**Scale**: Spotify processes over **600 billion events per day** through their pipelines, feeding into personalization algorithms like Discover Weekly.

### Uber's Data Pipeline

- **Sources**: Ride events, driver locations, payment transactions, ratings
- **Ingestion**: Apache Kafka (trillions of messages/day)
- **Processing**: Apache Spark + Apache Flink
- **Storage**: HDFS + Apache Hudi (data lake with updates)
- **Serving**: Presto/Trino for ad-hoc queries, BigQuery for BI
- **Orchestration**: Custom Airflow with 100K+ DAGs

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| No idempotency in pipelines | Re-running after failure creates duplicate data | Use UPSERT/MERGE, or partition-based overwrites |
| Ignoring data quality | Bad data propagates, corrupts dashboards and ML models | Add validation checks (Great Expectations, dbt tests) |
| Not handling schema evolution | Source adds a column → pipeline breaks | Use schema-on-read, Avro schemas, or flexible ingestion tools |
| Monolithic pipeline (one giant script) | Failure means re-running everything | Break into small, independent tasks with clear dependencies |
| No monitoring/alerting | Silent failures — stale data for days before anyone notices | Alert on pipeline SLA breaches, row count anomalies |
| Tight coupling to source schema | Source team changes table → pipeline breaks | Use contracts, versioned APIs, or CDC with schema registry |

---

## When to Use / When NOT to Use

### Use ETL When:
- ✅ Regulatory requirements demand only clean data in the warehouse
- ✅ Network bandwidth between source and warehouse is limited (transform reduces size)
- ✅ On-premise data warehouse with limited compute
- ✅ Well-established, rarely-changing transformation logic

### Use ELT When:
- ✅ Using cloud warehouses (Snowflake, BigQuery, Redshift)
- ✅ Transformation logic changes frequently
- ✅ Multiple teams need different views of the same raw data
- ✅ You want to keep raw data for future use cases not yet imagined
- ✅ You want faster time-to-insight (load first, ask questions later)

### Use Streaming Pipelines When:
- ✅ You need data freshness of seconds/minutes (not hours)
- ✅ Real-time fraud detection, live dashboards, alerting
- ✅ Event-driven architectures where downstream systems react immediately

### Avoid Over-Engineering When:
- ❌ You have less than 1 million rows total — a simple cron + SQL script is fine
- ❌ You're building pipelines for data nobody will use
- ❌ You're adding streaming when hourly batch is perfectly acceptable

---

## Key Takeaways

1. **Data pipelines** move data from operational systems (OLTP) to analytical systems (OLAP) through Extract, Transform, and Load steps.

2. **ETL** transforms data *before* loading (traditional). **ELT** loads raw data first, then transforms *inside* the warehouse (modern).

3. The **Modern Data Stack** (Fivetran + dbt + Snowflake) has made ELT the default for most companies because cloud warehouses have cheap storage and elastic compute.

4. **Batch pipelines** (Airflow, daily jobs) are simpler but introduce hours of latency. **Streaming pipelines** (Kafka + Flink) provide seconds of latency but are much more complex.

5. **Idempotency** is critical — you must be able to re-run any pipeline step without creating duplicates.

6. **Data quality** checks are not optional. Bad data in → bad decisions out.

7. The best pipeline is the **simplest one that meets your latency requirements**. Don't use streaming if daily batch is good enough.

---

## What's Next?

Now that you understand how data flows from OLTP to OLAP, let's explore **where** that analytical data actually lives. In [Chapter 20.3: Data Lakes & Data Warehouses](./03-data-lakes-warehouses.md), we'll compare S3-based data lakes, cloud warehouses like BigQuery and Snowflake, and understand when to use each.
