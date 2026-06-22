# 🔄 Chapter 5.1 — ETL/ELT — Data Pipelines Fundamentals

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Part 1 Foundations, Basic SQL Knowledge

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **ETL vs ELT** — and why this difference matters in 2026
- Build mental models for **data pipeline architecture**
- Know the **top tools**: Apache Airflow, dbt, Talend, SSIS, Fivetran, Airbyte
- Design pipelines that are **reliable, idempotent, and observable**
- Handle real-world challenges: **schema drift, late-arriving data, error handling**
- Think like a **data engineer** — not just a database user

---

## 🧠 The Problem — Why Do Data Pipelines Exist?

Imagine you're the CTO of an e-commerce company. Your data lives in **5 different places**:

```
                    The Data Chaos Problem

    ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐
    │ Website  │  │  Mobile  │  │  Payment  │  │ Inventory │  │  CRM     │
    │ (PG SQL) │  │ (MongoDB)│  │ (Stripe)  │  │  (MySQL)  │  │(Salesfrc)│
    └──────────┘  └──────────┘  └───────────┘  └───────────┘  └──────────┘

    CEO asks: "What's our revenue by product category per region this quarter?"

    ❌ Without pipelines:
       → Manually export CSVs from 5 systems
       → Open Excel, vlookup, pray it works
       → Takes 3 days, full of errors
       → By the time you finish, the data is stale 💀

    ✅ With data pipelines:
       → Automated ETL/ELT pulls from all 5 sources
       → Transforms, cleans, and loads into a Data Warehouse
       → CEO opens a dashboard → answer in 2 seconds ⚡
       → Runs every hour automatically
```

> **A data pipeline is an automated process that moves data from source systems to destination systems, transforming it along the way.**

---

## 📦 ETL — Extract, Transform, Load (The Classic Approach)

### The 3 Steps

```
                        ETL Pipeline Flow

    ┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
    │  EXTRACT    │────►│   TRANSFORM      │────►│     LOAD        │
    │             │     │                  │     │                 │
    │ Pull data   │     │ Clean, reshape,  │     │ Push to target  │
    │ from source │     │ enrich, validate │     │ (warehouse/DB)  │
    │             │     │                  │     │                 │
    │ • Databases │     │ • Filter nulls   │     │ • Data Warehouse│
    │ • APIs      │     │ • Join datasets  │     │ • Data Lake     │
    │ • Files     │     │ • Apply rules    │     │ • Analytics DB  │
    │ • Streams   │     │ • Aggregate      │     │ • Search Index  │
    └─────────────┘     └──────────────────┘     └─────────────────┘
                               ↑
                    Transformation happens
                    BEFORE loading (on ETL server)
```

### Real-World ETL Example

```
Scenario: Load daily sales into a data warehouse

EXTRACT:
  → Connect to production PostgreSQL
  → SELECT * FROM orders WHERE created_at >= '2026-06-01'
  → Pull 50,000 new orders

TRANSFORM (on ETL server):
  → Remove test/internal orders (WHERE email NOT LIKE '%@company.com')
  → Convert all currencies to USD using exchange rate table
  → Enrich with customer segment (JOIN with CRM data)
  → Calculate derived metrics (total_with_tax = amount * 1.08)
  → Validate: reject rows where amount < 0

LOAD:
  → Bulk INSERT into Snowflake fact_orders table
  → Update dimension tables (dim_customer, dim_product)
  → Log: "50,000 rows extracted, 48,723 transformed, 48,723 loaded"
```

---

## ⚡ ELT — Extract, Load, Transform (The Modern Approach)

### The Key Difference

```
                        ELT Pipeline Flow

    ┌─────────────┐     ┌─────────────────┐     ┌──────────────────┐
    │  EXTRACT    │────►│     LOAD         │────►│   TRANSFORM      │
    │             │     │                  │     │                  │
    │ Pull raw    │     │ Dump raw data    │     │ Transform INSIDE │
    │ data from   │     │ into target      │     │ the warehouse    │
    │ sources     │     │ warehouse AS-IS  │     │ using SQL/dbt    │
    └─────────────┘     └─────────────────┘     └──────────────────┘
                                                        ↑
                                              Transformation happens
                                              INSIDE the warehouse
                                              (using warehouse compute)
```

### Why ELT Is Taking Over

```
  2010s (ETL Era):
  ┌──────────────────────────────────────────────────────┐
  │  Warehouses were EXPENSIVE (Oracle, Teradata)        │
  │  → Transform BEFORE loading to minimize storage      │
  │  → ETL servers do heavy lifting                      │
  │  → Warehouse only sees clean data                    │
  └──────────────────────────────────────────────────────┘

  2020s+ (ELT Era):
  ┌──────────────────────────────────────────────────────┐
  │  Cloud warehouses are CHEAP & POWERFUL                │
  │  (BigQuery, Snowflake, Redshift, Databricks)         │
  │  → Storage costs pennies per GB                      │
  │  → Compute scales elastically                        │
  │  → Load raw data FAST, transform later with SQL      │
  │  → Keep raw data for re-transformation               │
  └──────────────────────────────────────────────────────┘
```

---

## 🔬 ETL vs ELT — Complete Comparison

| Dimension | ETL | ELT |
|-----------|-----|-----|
| **Transform Location** | Separate ETL server | Inside the data warehouse |
| **Data Loaded** | Clean, transformed data | Raw data (then transformed) |
| **Cost Model** | ETL server compute costs | Warehouse compute costs |
| **Raw Data Retention** | Often lost after transform | Always available for re-processing |
| **Flexibility** | Rigid — change = rebuild pipeline | Flexible — re-transform anytime |
| **Latency** | Higher (transform before load) | Lower (load immediately) |
| **Best For** | On-prem, sensitive data, legacy systems | Cloud-native, analytics, agile teams |
| **Compliance** | Easier (PII removed before warehouse) | Need extra PII handling in warehouse |
| **Skills Required** | Java/Python/ETL tool expertise | SQL-heavy (dbt, warehouse SQL) |
| **Scalability** | Limited by ETL server | Scales with warehouse compute |
| **Example Tools** | SSIS, Informatica, Talend, Pentaho | dbt, Fivetran + Snowflake, Airbyte |
| **Debugging** | Harder (transform is a black box) | Easier (all data visible in warehouse) |

### 💡 Pro Tip — The Real World Uses Both

```
Modern Architecture (Hybrid):

  Sources ──► Fivetran/Airbyte (EL) ──► Raw Layer (Data Lake/Warehouse)
                                              │
                                              ▼
                                      dbt (Transform in SQL)
                                              │
                                    ┌─────────┴──────────┐
                                    ▼                    ▼
                              Staging Layer        Mart Layer
                              (cleaned data)       (business-ready)
                                                        │
                                                        ▼
                                                  Dashboards / ML
```

---

## 🏗️ Data Pipeline Architecture Patterns

### Pattern 1: Batch Pipeline (Most Common)

```
  Runs on a SCHEDULE (hourly, daily, weekly)

  ┌─────────┐    ┌───────┐    ┌───────────┐    ┌─────────────┐
  │  Source  │───►│Extract│───►│ Transform │───►│    Load      │
  │  DB     │    │ (full │    │ (clean +  │    │ (bulk insert │
  │         │    │  or   │    │  enrich)  │    │  or merge)   │
  └─────────┘    │ incr) │    └───────────┘    └─────────────┘
                 └───────┘
                     │
                     ▼
              Schedule: Every day at 2 AM
              Data freshness: T+1 (yesterday's data)

  Use cases: Daily reports, weekly aggregations, monthly billing
```

### Pattern 2: Micro-Batch Pipeline

```
  Runs every FEW MINUTES (5-15 min intervals)

  Source ──► CDC (Change Data Capture) ──► Buffer ──► Transform ──► Load
                                           (Kafka)

  Schedule: Every 5 minutes
  Data freshness: T+5min (near real-time)

  Use cases: Dashboards, fraud scoring, inventory updates
```

### Pattern 3: Real-Time / Streaming Pipeline

```
  Processes data AS IT ARRIVES (millisecond latency)

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌───────────┐
  │  Event   │───►│  Stream  │───►│ Stream   │───►│  Target   │
  │ Producer │    │ Platform │    │ Processor│    │  (DB/     │
  │ (app)    │    │ (Kafka)  │    │ (Flink/  │    │  Dashboard│
  │          │    │          │    │  Spark)  │    │  /Alert)  │
  └──────────┘    └──────────┘    └──────────┘    └───────────┘

  Latency: Milliseconds to seconds
  Data freshness: Real-time

  Use cases: Fraud detection, stock trading, live dashboards, IoT
```

### When to Use Which Pattern

| Pattern | Latency | Complexity | Cost | Best For |
|---------|---------|-----------|------|----------|
| Batch | Hours | Low | Low | Reports, analytics, ML training |
| Micro-Batch | Minutes | Medium | Medium | Dashboards, monitoring |
| Streaming | Seconds/ms | High | High | Fraud, trading, IoT, alerts |

> 💡 **Start with batch. Move to micro-batch. Go streaming ONLY when business requires it.**

---

## 🛠️ The ETL/ELT Tool Landscape

### 1. Apache Airflow — The Orchestrator King 🔥

> **What:** An open-source workflow orchestration platform
> **Written in:** Python
> **Used by:** Airbnb (created it), Slack, Robinhood, Adobe, Lyft

```python
# Airflow DAG Example — Daily Sales Pipeline
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['data-alerts@company.com'],
}

with DAG(
    dag_id='daily_sales_pipeline',
    default_args=default_args,
    description='Extract sales data, transform, load to warehouse',
    schedule_interval='0 2 * * *',  # Every day at 2 AM
    start_date=datetime(2026, 1, 1),
    catchup=False,               # Don't backfill missed runs
    tags=['sales', 'daily'],
) as dag:

    extract_orders = PythonOperator(
        task_id='extract_orders',
        python_callable=extract_from_postgres,   # Your function
    )

    extract_customers = PythonOperator(
        task_id='extract_customers',
        python_callable=extract_from_crm,
    )

    transform_data = PythonOperator(
        task_id='transform_and_enrich',
        python_callable=transform_sales_data,
    )

    load_to_warehouse = PostgresOperator(
        task_id='load_to_warehouse',
        postgres_conn_id='warehouse_conn',
        sql='sql/load_fact_sales.sql',
    )

    run_quality_checks = PythonOperator(
        task_id='data_quality_checks',
        python_callable=run_dq_checks,
    )

    notify_team = PythonOperator(
        task_id='notify_completion',
        python_callable=send_slack_notification,
    )

    # Define dependencies (this creates the DAG shape)
    [extract_orders, extract_customers] >> transform_data >> load_to_warehouse
    load_to_warehouse >> run_quality_checks >> notify_team
```

```
  Airflow DAG Visualization:

  extract_orders ──────┐
                       ├──► transform_data ──► load_to_warehouse ──► quality_checks ──► notify
  extract_customers ───┘

  Key Concepts:
  ├── DAG (Directed Acyclic Graph) = The pipeline definition
  ├── Task = A single unit of work (extract, transform, etc.)
  ├── Operator = Template for a task (PythonOperator, SQLOperator)
  ├── Schedule = When to run (cron expression)
  ├── Sensor = Wait for a condition (file arrival, API response)
  └── XCom = Pass small data between tasks
```

#### Airflow Pros & Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Python-native — write DAGs as code | Learning curve for beginners |
| Massive community & provider ecosystem | Not designed for streaming |
| Rich UI (task logs, Gantt charts, tree view) | Can be resource-heavy |
| Battle-tested at massive scale | Scheduler can be a bottleneck |
| Managed options: MWAA (AWS), Cloud Composer (GCP) | DAGs can get complex fast |

---

### 2. dbt (Data Build Tool) — The T in ELT 🔥🔥

> **What:** Transforms data already in your warehouse using SQL + software engineering best practices
> **Philosophy:** "Analytics is code" — version control, testing, documentation for SQL
> **Used by:** GitLab, JetBlue, Hubspot, thousands of data teams

```sql
-- dbt Model Example: models/marts/fact_orders.sql

{{ config(
    materialized='incremental',
    unique_key='order_id',
    schema='marts'
) }}

WITH raw_orders AS (
    SELECT * FROM {{ source('ecommerce', 'orders') }}
    {% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
    {% endif %}
),

enriched_orders AS (
    SELECT
        o.order_id,
        o.customer_id,
        c.customer_segment,
        c.country,
        o.product_id,
        p.category AS product_category,
        o.quantity,
        o.unit_price,
        o.quantity * o.unit_price AS total_amount,
        o.currency,
        CASE
            WHEN o.currency != 'USD'
            THEN o.quantity * o.unit_price * fx.rate
            ELSE o.quantity * o.unit_price
        END AS total_amount_usd,
        o.order_status,
        o.created_at,
        o.updated_at
    FROM raw_orders o
    LEFT JOIN {{ ref('dim_customers') }} c ON o.customer_id = c.customer_id
    LEFT JOIN {{ ref('dim_products') }} p ON o.product_id = p.product_id
    LEFT JOIN {{ ref('exchange_rates') }} fx
        ON o.currency = fx.currency AND o.created_at::date = fx.rate_date
)

SELECT * FROM enriched_orders
```

```yaml
# dbt Test Example: models/marts/schema.yml
version: 2

models:
  - name: fact_orders
    description: "Clean, enriched orders fact table"
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: total_amount_usd
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 1000000
      - name: customer_segment
        tests:
          - accepted_values:
              values: ['Enterprise', 'SMB', 'Consumer', 'Unknown']
```

```
  dbt Project Structure:

  my_dbt_project/
  ├── dbt_project.yml          # Project config
  ├── models/
  │   ├── staging/             # 1:1 with source tables (clean + rename)
  │   │   ├── stg_orders.sql
  │   │   ├── stg_customers.sql
  │   │   └── schema.yml       # Tests & docs for staging models
  │   ├── intermediate/        # Business logic, joins
  │   │   └── int_orders_enriched.sql
  │   └── marts/               # Final tables for consumption
  │       ├── fact_orders.sql
  │       ├── dim_customers.sql
  │       └── schema.yml
  ├── seeds/                   # CSV files loaded as tables (country codes, etc.)
  ├── snapshots/               # Track changes over time (SCD Type 2)
  ├── macros/                  # Reusable SQL snippets (Jinja)
  └── tests/                   # Custom data quality tests
```

#### dbt Materializations

| Type | What It Does | When to Use |
|------|-------------|-------------|
| `view` | Creates a SQL VIEW | Lightweight, always fresh, small data |
| `table` | Creates a TABLE (drops + recreates) | Medium data, OK to rebuild |
| `incremental` | Only processes NEW rows | Large tables, append-heavy |
| `ephemeral` | CTE injected into downstream models | Intermediate calculations |
| `snapshot` | Tracks row changes over time (SCD2) | Slowly Changing Dimensions |

---

### 3. Fivetran / Airbyte — The EL (Extract + Load) Tools

```
  Fivetran / Airbyte Architecture:

  ┌──────────────┐     ┌───────────────────┐     ┌────────────────┐
  │  Sources     │     │  Fivetran/Airbyte  │     │  Destinations  │
  │              │────►│                   │────►│                │
  │ • PostgreSQL │     │ • Pre-built        │     │ • Snowflake    │
  │ • MySQL      │     │   connectors       │     │ • BigQuery     │
  │ • Salesforce │     │ • CDC replication   │     │ • Redshift     │
  │ • Stripe     │     │ • Schema detection  │     │ • Databricks   │
  │ • HubSpot   │     │ • Incremental sync  │     │ • PostgreSQL   │
  │ • MongoDB    │     │ • Normalization     │     │                │
  │ • 300+ more  │     │                   │     │                │
  └──────────────┘     └───────────────────┘     └────────────────┘

  Fivetran: Fully managed SaaS (paid)     → Zero code, just configure
  Airbyte:  Open-source (self-host/cloud) → Free, community connectors
```

| Feature | Fivetran | Airbyte |
|---------|----------|---------|
| **Pricing** | Per-row pricing ($$) | Open-source (free self-host) |
| **Connectors** | 300+ (enterprise quality) | 350+ (community maintained) |
| **Setup Time** | Minutes | Minutes (cloud) / Hours (self-host) |
| **Reliability** | Enterprise-grade SLA | Depends on connector quality |
| **CDC Support** | Excellent | Good (improving) |
| **Custom Connectors** | Limited | Build your own easily |

---

### 4. SSIS — SQL Server Integration Services

> **What:** Microsoft's ETL tool, deeply integrated with SQL Server
> **Best For:** Microsoft-stack enterprises

```
  SSIS Architecture:

  ┌──────────────────────────────────────────────────────┐
  │                  SSIS Package                        │
  │                                                      │
  │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
  │  │ Control Flow │  │ Control Flow │  │Control Flow│ │
  │  │  Task 1      │─►│  Task 2     │─►│  Task 3    │ │
  │  │ (Execute SQL)│  │(Data Flow)  │  │(Send Mail) │ │
  │  └─────────────┘  └──────┬───────┘  └────────────┘ │
  │                          │                           │
  │                    ┌─────▼──────┐                    │
  │                    │ Data Flow  │                    │
  │                    │            │                    │
  │                    │ Source ──► │                    │
  │                    │ Transform─►│                    │
  │                    │ Destination│                    │
  │                    └────────────┘                    │
  └──────────────────────────────────────────────────────┘

  Control Flow: WHAT to do (sequence of tasks)
  Data Flow:    HOW data moves (row-by-row transformations)
```

---

### 5. Other Notable Tools

| Tool | Type | Best For | Key Feature |
|------|------|----------|-------------|
| **Apache Spark** | Distributed processing | Big data (TB+) | In-memory computing, PySpark |
| **Apache Kafka** | Streaming platform | Real-time event streaming | Pub/sub, exactly-once delivery |
| **Apache Flink** | Stream processing | Real-time analytics | True streaming (not micro-batch) |
| **Talend** | ETL/ELT | Enterprise integration | Visual designer, 900+ connectors |
| **Informatica** | ETL/ELT | Large enterprise | Industry veteran, AI-powered |
| **Pentaho** | ETL | SMB / open-source | Free community edition |
| **Dagster** | Orchestration | Modern data teams | Software-defined assets |
| **Prefect** | Orchestration | Python-native teams | Simpler than Airflow |
| **Meltano** | EL | Singer-based extraction | CLI-first, GitOps friendly |
| **AWS Glue** | Serverless ETL | AWS ecosystem | Serverless Spark, crawlers |

---

## 🔑 Critical Pipeline Concepts Every Engineer Must Know

### 1. Full Load vs Incremental Load

```
  FULL LOAD:
  ┌────────────────────┐          ┌─────────────────────┐
  │ Source (10M rows)  │ ════════►│ Target (10M rows)   │
  └────────────────────┘          │ (DROP + RECREATE)   │
                                  └─────────────────────┘
  • Copies EVERYTHING every time
  • Simple but SLOW for large tables
  • Use for: small tables, dimension tables, first-time load

  INCREMENTAL LOAD:
  ┌────────────────────┐          ┌─────────────────────┐
  │ Source (10M rows)  │          │ Target (10M rows)   │
  │                    │          │                     │
  │ New: 5,000 rows ──┼─────────►│ INSERT 5,000        │
  │ Changed: 200 rows ┼─────────►│ UPDATE 200          │
  └────────────────────┘          └─────────────────────┘
  • Only processes CHANGES since last run
  • Fast but needs a "watermark" (timestamp, ID, CDC)
  • Use for: large fact tables, event logs, transaction tables
```

#### Incremental Strategies

```sql
-- Strategy 1: Timestamp-based (most common)
SELECT * FROM orders
WHERE updated_at > '2026-06-01 02:00:00'   -- Last successful run time

-- Strategy 2: Auto-increment ID based
SELECT * FROM orders
WHERE order_id > 4500000                    -- Last loaded ID

-- Strategy 3: CDC (Change Data Capture) — the gold standard
-- Database captures every INSERT, UPDATE, DELETE as events
-- Tools: Debezium, Oracle GoldenGate, SQL Server CDC, Maxwell
```

### 2. Idempotency — The #1 Pipeline Design Rule

```
  IDEMPOTENT: Running a pipeline MULTIPLE times produces the SAME result.

  ❌ NOT Idempotent (DANGER):
     INSERT INTO target SELECT * FROM source WHERE date = '2026-06-01';
     -- Run once: 1,000 rows
     -- Run again (retry): 2,000 rows  💀 DUPLICATES!

  ✅ Idempotent (SAFE):
     -- Option 1: DELETE + INSERT
     DELETE FROM target WHERE date = '2026-06-01';
     INSERT INTO target SELECT * FROM source WHERE date = '2026-06-01';
     -- Run once: 1,000 rows
     -- Run again: Still 1,000 rows ✅

     -- Option 2: MERGE / UPSERT
     MERGE INTO target t
     USING source s ON t.order_id = s.order_id
     WHEN MATCHED THEN UPDATE SET ...
     WHEN NOT MATCHED THEN INSERT ...;
     -- Run once or 100 times: Same result ✅
```

> 💡 **Golden Rule: Pipelines WILL fail and WILL be re-run. Design for it from Day 1.**

### 3. Data Quality & Validation

```
                  Data Quality Pyramid

            ┌────────────────────────┐
            │    BUSINESS RULES      │  "Revenue should never
            │    (Semantic)          │   be negative"
            ├────────────────────────┤
            │    REFERENTIAL         │  "Every order_id must
            │    INTEGRITY           │   have a valid customer"
            ├────────────────────────┤
            │    CONSISTENCY         │  "Dates should be
            │    CHECKS              │   ISO 8601 format"
            ├────────────────────────┤
            │    COMPLETENESS        │  "email column should
            │    CHECKS              │   be < 5% null"
            ├────────────────────────┤
            │    FRESHNESS           │  "Data should be no
            │    CHECKS              │   older than 2 hours"
            ├────────────────────────┤
            │    VOLUME / ROW        │  "Daily orders should
            │    COUNT CHECKS        │   be between 1K–50K"
            └────────────────────────┘
```

```python
# Great Expectations — Data Quality Framework
import great_expectations as gx

context = gx.get_context()

# Define expectations for the orders table
validator = context.sources.pandas_default.read_csv("orders.csv")

validator.expect_column_values_to_not_be_null("order_id")
validator.expect_column_values_to_be_unique("order_id")
validator.expect_column_values_to_be_between("total_amount", min_value=0, max_value=1000000)
validator.expect_column_values_to_be_in_set("status", ["pending", "shipped", "delivered", "cancelled"])
validator.expect_table_row_count_to_be_between(min_value=1000, max_value=100000)

result = validator.validate()
if not result.success:
    raise Exception("Data quality check FAILED! Pipeline halted.")
```

### 4. Schema Drift — When Sources Change Without Warning

```
  The Nightmare Scenario:

  Monday:    Source table has columns: [id, name, email, phone]
  Tuesday:   Dev adds column:          [id, name, email, phone, address]
  Wednesday: Pipeline BREAKS 💥 because target doesn't have 'address'

  Solutions:
  ┌──────────────────────────────────────────────────────────────┐
  │  1. Schema Evolution (Auto-adapt)                            │
  │     → Tools like Fivetran auto-add new columns               │
  │     → Airbyte can detect schema changes                      │
  │                                                              │
  │  2. Schema Contract (Strict)                                 │
  │     → Define expected schema in code/config                  │
  │     → Pipeline fails LOUDLY if schema changes                │
  │     → Requires manual review before proceeding               │
  │                                                              │
  │  3. Schema Registry (Kafka/Avro)                             │
  │     → Centralized schema definitions                         │
  │     → Backward/forward compatibility rules                   │
  │     → Producers can't break consumers                        │
  └──────────────────────────────────────────────────────────────┘
```

### 5. Slowly Changing Dimensions (SCD)

```
  Customer "John" changes his city from "NYC" to "LA".
  How do you handle this in the warehouse?

  SCD Type 1 — Overwrite (No History)
  ┌──────────────────────────────────────────┐
  │  customer_id │ name │ city │             │
  │  101         │ John │ LA   │  ← NYC gone │
  └──────────────────────────────────────────┘

  SCD Type 2 — Add New Row (Full History) ⭐ Most Common
  ┌───────────────────────────────────────────────────────────────┐
  │  surrogate_key │ customer_id │ name │ city │ valid_from │ valid_to   │ is_current │
  │  1001          │ 101         │ John │ NYC  │ 2024-01-01 │ 2026-05-31 │ false      │
  │  1002          │ 101         │ John │ LA   │ 2026-06-01 │ 9999-12-31 │ true       │
  └───────────────────────────────────────────────────────────────┘

  SCD Type 3 — Add Column (Limited History)
  ┌──────────────────────────────────────────────────────┐
  │  customer_id │ name │ current_city │ previous_city   │
  │  101         │ John │ LA           │ NYC             │
  └──────────────────────────────────────────────────────┘
```

```sql
-- dbt Snapshot for SCD Type 2 (automatic!)
-- snapshots/snap_customers.sql

{% snapshot snap_customers %}
    {{
        config(
          target_schema='snapshots',
          unique_key='customer_id',
          strategy='timestamp',
          updated_at='updated_at',
        )
    }}

    SELECT * FROM {{ source('crm', 'customers') }}

{% endsnapshot %}
-- dbt automatically tracks changes, adds valid_from/valid_to/is_current!
```

---

## 🏗️ Building a Modern Data Pipeline — End-to-End Example

```
                    Modern ELT Architecture

  ┌─────────────────────────────────────────────────────────────────┐
  │                     SOURCE SYSTEMS                              │
  ├───────────┬───────────┬───────────┬───────────┬────────────────┤
  │ PostgreSQL│  MongoDB  │  Stripe   │ Salesforce│  Google Ads    │
  │ (app DB)  │ (events)  │ (payments)│   (CRM)   │  (marketing)   │
  └─────┬─────┴─────┬─────┴─────┬─────┴─────┬─────┴───────┬────────┘
        │           │           │           │             │
        ▼           ▼           ▼           ▼             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │              INGESTION LAYER (Airbyte / Fivetran)               │
  │              CDC + API connectors → raw data replication         │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │              STORAGE LAYER (Snowflake / BigQuery)               │
  │                                                                 │
  │  ┌──────────┐   ┌──────────────┐   ┌─────────────────────────┐│
  │  │ RAW      │──►│   STAGING    │──►│         MARTS           ││
  │  │ (as-is)  │   │  (cleaned)   │   │   (business-ready)      ││
  │  │          │   │              │   │                         ││
  │  │raw.orders│   │stg_orders    │   │ fact_orders             ││
  │  │raw.users │   │stg_customers │   │ dim_customers           ││
  │  │raw.events│   │stg_payments  │   │ dim_products            ││
  │  └──────────┘   └──────────────┘   │ mart_revenue_by_region  ││
  │                                     └─────────────────────────┘│
  │              ▲ dbt transforms (SQL + Jinja) ▲                  │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │              CONSUMPTION LAYER                                   │
  │                                                                  │
  │  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌───────────────┐ │
  │  │ Looker / │  │ Jupyter / │  │  Reverse  │  │  ML Models    │ │
  │  │ Tableau  │  │ Notebooks │  │   ETL     │  │  (training)   │ │
  │  │(dashbrd) │  │(analysis) │  │(push to   │  │               │ │
  │  │          │  │           │  │ Salesforce)│  │               │ │
  │  └──────────┘  └───────────┘  └───────────┘  └───────────────┘ │
  └──────────────────────────────────────────────────────────────────┘

  ORCHESTRATION: Apache Airflow / Dagster (schedules + monitors everything)
```

---

## ⚠️ Common Pipeline Anti-Patterns (Avoid These!)

| Anti-Pattern | Why It's Bad | What to Do Instead |
|---|---|---|
| **No idempotency** | Retries create duplicate data | Use MERGE/UPSERT or DELETE+INSERT |
| **Hardcoded credentials** | Security nightmare | Use secrets manager (Vault, AWS SSM) |
| **No monitoring/alerting** | Silent failures for days | Alert on failures, data freshness, row counts |
| **Monolithic pipelines** | One failure kills everything | Break into small, independent DAGs |
| **Processing all data every run** | Wastes time and compute | Use incremental loading |
| **No data quality checks** | Garbage in, garbage out | Add validation after every stage |
| **No schema versioning** | Breaking changes crash pipelines | Use migration tools, schema contracts |
| **Tight coupling to sources** | Source changes = pipeline breaks | Add a staging/raw layer as buffer |
| **No backfill support** | Can't reprocess historical data | Design pipelines with date ranges |
| **Logging to console only** | Can't debug production issues | Structured logging to persistent store |

---

## 🔄 Change Data Capture (CDC) — The Gold Standard for Syncing

```
  CDC captures EVERY change (INSERT, UPDATE, DELETE) from the database
  transaction log and streams it as events.

  ┌──────────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Source DB    │    │Debezium  │    │  Kafka   │    │ Target   │
  │              │    │(CDC Tool)│    │          │    │ System   │
  │  WAL / Redo │───►│          │───►│  Topic   │───►│          │
  │  Log        │    │ Reads tx │    │          │    │ Consumer │
  │              │    │ log      │    │          │    │          │
  └──────────────┘    └──────────┘    └──────────┘    └──────────┘

  Debezium CDC Event (JSON):
  {
    "op": "u",                    // c=create, u=update, d=delete
    "before": {                   // Previous state
      "id": 1001,
      "status": "pending",
      "amount": 99.99
    },
    "after": {                    // New state
      "id": 1001,
      "status": "shipped",
      "amount": 99.99
    },
    "source": {
      "db": "ecommerce",
      "table": "orders",
      "lsn": "0/1A2B3C4D"
    },
    "ts_ms": 1717286400000
  }
```

### CDC Tools Comparison

| Tool | Supported Sources | Streaming To | License |
|------|-------------------|-------------|---------|
| **Debezium** | PG, MySQL, MongoDB, Oracle, SQL Server | Kafka | Open Source |
| **AWS DMS** | All major DBs | S3, Kinesis, Redshift | AWS Managed |
| **Oracle GoldenGate** | Oracle, SQL Server, MySQL | Various | Commercial |
| **SQL Server CDC** | SQL Server only | Native | Built-in |
| **Maxwell** | MySQL only | Kafka, Kinesis, SQS | Open Source |
| **Fivetran CDC** | Many databases | Cloud warehouses | SaaS |

---

## 🧪 Hands-On: Build Your First Pipeline (Airflow + dbt)

### Step 1: Define the DAG in Airflow

```python
# dags/ecommerce_pipeline.py
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.providers.airbyte.operators.airbyte import AirbyteTriggerSyncOperator
from datetime import datetime

with DAG(
    'ecommerce_elt_pipeline',
    schedule_interval='@daily',
    start_date=datetime(2026, 1, 1),
    catchup=False,
) as dag:

    # Step 1: Extract + Load (via Airbyte)
    sync_postgres = AirbyteTriggerSyncOperator(
        task_id='sync_postgres_to_warehouse',
        airbyte_conn_id='airbyte_default',
        connection_id='postgres-to-snowflake-conn-id',
    )

    # Step 2: Transform (via dbt)
    dbt_run = BashOperator(
        task_id='dbt_run',
        bash_command='cd /opt/dbt/ecommerce && dbt run --select tag:daily',
    )

    # Step 3: Test (via dbt)
    dbt_test = BashOperator(
        task_id='dbt_test',
        bash_command='cd /opt/dbt/ecommerce && dbt test --select tag:daily',
    )

    sync_postgres >> dbt_run >> dbt_test
```

### Step 2: dbt Models (Transform Layer)

```sql
-- models/staging/stg_orders.sql
-- Clean raw orders: rename columns, cast types, filter junk

SELECT
    id AS order_id,
    user_id AS customer_id,
    CAST(total_cents AS DECIMAL(10,2)) / 100 AS total_amount,
    LOWER(status) AS order_status,
    CAST(created_at AS TIMESTAMP) AS ordered_at,
    CAST(updated_at AS TIMESTAMP) AS updated_at
FROM {{ source('raw', 'orders') }}
WHERE status != 'test'
  AND total_cents > 0
```

```sql
-- models/marts/mart_daily_revenue.sql
-- Business-ready: daily revenue by product category

{{ config(materialized='table') }}

SELECT
    DATE_TRUNC('day', o.ordered_at) AS order_date,
    p.category AS product_category,
    c.country,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(o.total_amount) AS total_revenue,
    AVG(o.total_amount) AS avg_order_value
FROM {{ ref('stg_orders') }} o
JOIN {{ ref('dim_products') }} p ON o.product_id = p.product_id
JOIN {{ ref('dim_customers') }} c ON o.customer_id = c.customer_id
GROUP BY 1, 2, 3
```

---

## 📊 Quick Reference — Pipeline Design Checklist

```
  Before shipping ANY pipeline to production:

  ✅ Idempotent?          Can I safely re-run without duplicates?
  ✅ Incremental?         Am I only processing new/changed data?
  ✅ Monitored?           Will I be alerted if it fails?
  ✅ Data quality checks? Am I validating output data?
  ✅ Backfill-able?       Can I reprocess historical data?
  ✅ Secrets secured?     Are credentials in a secrets manager?
  ✅ Documented?          Can a new engineer understand this pipeline?
  ✅ Tested?              Unit tests for transforms? Integration tests?
  ✅ Schema-aware?        Will I handle schema changes gracefully?
  ✅ Logged?              Can I trace every row from source to target?
```

---

## 🏆 Key Takeaways

| Concept | Remember This |
|---------|---------------|
| **ETL** | Transform before loading — classic, on-prem |
| **ELT** | Load raw, transform in warehouse — modern, cloud |
| **Airflow** | Orchestrate WHEN things run (DAGs) |
| **dbt** | Define HOW to transform (SQL models) |
| **Fivetran/Airbyte** | Handle WHAT to extract + load (connectors) |
| **CDC** | Capture changes from DB transaction logs |
| **Idempotency** | Pipeline can re-run safely — **non-negotiable** |
| **Incremental** | Only process changes — saves time and money |
| **Data Quality** | Validate at every stage — never trust source data |

---

> 🚀 **Next Up:** [Chapter 5.2 — Database Migration Strategies](./02-Database-Migration.md) — Learn how to evolve your database schema safely with Flyway, Liquibase, and zero-downtime migration techniques.

---

*Last Updated: June 2026*
