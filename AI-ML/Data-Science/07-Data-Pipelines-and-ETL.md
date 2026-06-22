# Chapter 07: Data Pipelines and ETL

## Table of Contents
- [What is ETL and Data Pipelines?](#what-is-etl-and-data-pipelines)
- [Why It Matters](#why-it-matters)
- [ETL vs ELT — The Core Difference](#etl-vs-elt--the-core-difference)
- [How Data Pipelines Work](#how-data-pipelines-work)
- [Data Warehousing Concepts](#data-warehousing-concepts)
- [Building ETL Pipelines in Python](#building-etl-pipelines-in-python)
- [Apache Airflow — Orchestrating Pipelines](#apache-airflow--orchestrating-pipelines)
- [Other Pipeline Tools](#other-pipeline-tools)
- [Data Quality and Validation](#data-quality-and-validation)
- [Streaming vs Batch Processing](#streaming-vs-batch-processing)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is ETL and Data Pipelines?

### Simple Explanation

Imagine you run a restaurant chain with 50 locations. Every day, each restaurant generates data: customer orders, inventory levels, staff hours, customer feedback. To make business decisions (like "which menu item is underperforming?"), you need to **collect** all that scattered data, **clean** it up (fix typos, remove duplicates, standardize formats), and **store** it in one organized place where analysts can query it.

**That process is ETL:**
- **E**xtract — Pull data from various sources (databases, APIs, files, sensors)
- **T**ransform — Clean, reshape, aggregate, validate the data
- **L**oad — Put the processed data into a destination (data warehouse, data lake)

A **Data Pipeline** is the automated system that does ETL on a schedule — like a factory assembly line for data.

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   EXTRACT   │────▶│  TRANSFORM   │────▶│    LOAD     │
│             │     │              │     │             │
│ • Databases │     │ • Clean      │     │ • Warehouse │
│ • APIs      │     │ • Validate   │     │ • Data Lake │
│ • Files     │     │ • Aggregate  │     │ • Dashboard │
│ • Streams   │     │ • Join       │     │ • ML Model  │
└─────────────┘     └──────────────┘     └─────────────┘
```

---

## Why It Matters

### Real-World Relevance

| Scenario | Without Pipelines | With Pipelines |
|----------|------------------|----------------|
| Daily sales report | Analyst manually exports CSVs from 5 systems, spends 3 hours cleaning | Report auto-generated at 6 AM, ready when team arrives |
| ML model retraining | Data scientist manually pulls fresh data each week | Model automatically retrains on fresh data nightly |
| Compliance audit | Panic mode — scrambling to find and organize historical data | All data versioned, traceable, and queryable |
| Real-time fraud detection | Impossible manually | Streaming pipeline flags suspicious transactions in <1 second |

### When You'd Use It

- **Always** in production data science — no company runs on Jupyter notebooks in production
- When data comes from multiple sources that need to be combined
- When data needs to be refreshed on a schedule (hourly, daily, weekly)
- When multiple teams/dashboards need the same cleaned data
- When you need audit trails and reproducibility

> **Important:** If you're a data scientist who can't build pipelines, you're limited to ad-hoc analysis. Pipeline skills are what separate "notebook data scientists" from production-ready ones.

---

## ETL vs ELT — The Core Difference

### ETL (Extract, Transform, Load)

```
Source → Transform OUTSIDE warehouse → Load clean data into warehouse
```

**Traditional approach.** Transform data before loading it. Used when:
- Storage is expensive (you only store what you need)
- Data warehouse has limited compute power
- You know exactly what transformations are needed upfront
- Compliance requires filtering sensitive data before it enters the warehouse

### ELT (Extract, Load, Transform)

```
Source → Load RAW data into warehouse → Transform INSIDE warehouse
```

**Modern approach.** Load everything raw first, transform later. Used when:
- Storage is cheap (cloud data lakes)
- Warehouse has powerful compute (BigQuery, Snowflake, Redshift)
- Requirements change frequently (raw data is always available)
- You want to support ad-hoc analysis on raw data

### Comparison Table

| Aspect | ETL | ELT |
|--------|-----|-----|
| Transform location | External server/tool | Inside the data warehouse |
| Storage cost | Lower (only clean data stored) | Higher (raw + transformed data) |
| Flexibility | Lower (must re-extract if needs change) | Higher (raw data always available) |
| Speed of initial load | Slower (transform first) | Faster (load raw, transform later) |
| Data freshness | May lag | Can be near-real-time |
| Best for | Structured, well-defined use cases | Exploratory, evolving use cases |
| Tools | Informatica, Talend, SSIS | dbt, Snowflake, BigQuery |

> **Pro Tip:** Most modern data stacks use ELT. The tool **dbt (data build tool)** has become the standard for the "T" in ELT — it lets you write transformations as SQL and version-control them.

---

## How Data Pipelines Work

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        DATA PIPELINE ARCHITECTURE                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  DATA SOURCES              INGESTION         PROCESSING     DESTINATION│
│  ───────────              ─────────         ──────────     ───────────│
│                                                                        │
│  ┌─────────┐            ┌──────────┐      ┌──────────┐   ┌─────────┐│
│  │ Database│──┐         │  Kafka/  │      │  Spark/  │   │Warehouse││
│  └─────────┘  │         │  Kinesis │      │  Pandas  │   └─────────┘│
│  ┌─────────┐  │  ┌───┐  └──────────┘      └──────────┘   ┌─────────┐│
│  │  API    │──┼─▶│ E │─────────────────────────────────▶  │Data Lake││
│  └─────────┘  │  └───┘  ┌──────────┐      ┌──────────┐   └─────────┘│
│  ┌─────────┐  │         │  Airflow │      │   dbt    │   ┌─────────┐│
│  │  Files  │──┘         │(schedule)│      │(SQL xfrm)│   │Dashboard││
│  └─────────┘            └──────────┘      └──────────┘   └─────────┘│
│                                                                        │
│  MONITORING: Alerts, Logging, Data Quality Checks                     │
└──────────────────────────────────────────────────────────────────────┘
```

### Pipeline Components

#### 1. Data Sources (Extract)
- **Databases:** PostgreSQL, MySQL, MongoDB, Oracle
- **APIs:** REST APIs, GraphQL endpoints
- **Files:** CSV, JSON, Parquet, Excel on S3/GCS/local
- **Streams:** Kafka topics, IoT sensors, clickstream data
- **SaaS tools:** Salesforce, HubSpot, Google Analytics

#### 2. Orchestration (Scheduling & Dependencies)
- Defines WHEN tasks run and in WHAT ORDER
- Handles retries, alerts on failure
- Tools: Airflow, Prefect, Dagster, Luigi

#### 3. Processing (Transform)
- Data cleaning, deduplication
- Joins across sources
- Aggregations (daily → weekly summaries)
- Business logic application
- Tools: Pandas, PySpark, dbt, SQL

#### 4. Storage (Load)
- **Data Warehouse:** Snowflake, BigQuery, Redshift (structured, query-optimized)
- **Data Lake:** S3, GCS, ADLS (raw, any format, cheap)
- **Data Lakehouse:** Databricks, Delta Lake (combines both)

#### 5. Monitoring & Alerting
- Did the pipeline run successfully?
- Is the data quality acceptable?
- Are there unexpected nulls, duplicates, or volume changes?

---

## Data Warehousing Concepts

### Star Schema vs Snowflake Schema

#### Star Schema
```
                    ┌──────────────┐
                    │  dim_product │
                    │──────────────│
                    │ product_id   │
                    │ name         │
                    │ category     │
                    └──────┬───────┘
                           │
┌──────────────┐    ┌──────┴───────┐    ┌──────────────┐
│  dim_time    │    │  fact_sales  │    │ dim_customer │
│──────────────│    │──────────────│    │──────────────│
│ date_id      │────│ date_id      │────│ customer_id  │
│ day          │    │ product_id   │    │ name         │
│ month        │    │ customer_id  │    │ segment      │
│ quarter      │    │ store_id     │    │ region       │
│ year         │    │ quantity     │    └──────────────┘
└──────────────┘    │ revenue      │
                    │ discount     │    ┌──────────────┐
                    │──────────────│────│  dim_store   │
                    └──────────────┘    │──────────────│
                                        │ store_id     │
                                        │ city         │
                                        │ state        │
                                        └──────────────┘
```

- **Fact Table:** Contains measurable events (sales, clicks, transactions)
- **Dimension Tables:** Contain descriptive attributes (who, what, where, when)
- **Star Schema:** Dimensions connect directly to fact table (simple, fast queries)
- **Snowflake Schema:** Dimensions are normalized (less redundancy, more joins)

### Data Warehouse vs Data Lake vs Data Lakehouse

| Feature | Data Warehouse | Data Lake | Data Lakehouse |
|---------|---------------|-----------|----------------|
| Data type | Structured only | Any (structured, semi, unstructured) | Any |
| Schema | Schema-on-write | Schema-on-read | Both |
| Cost | High | Low | Medium |
| Query speed | Fast | Slow without optimization | Fast |
| Users | Business analysts, BI | Data engineers, ML engineers | Everyone |
| Examples | Snowflake, BigQuery | S3, HDFS | Databricks, Delta Lake |

---

## Building ETL Pipelines in Python

### Basic ETL with Pandas

```python
import pandas as pd
import sqlite3
from datetime import datetime
import logging

# Set up logging — ALWAYS log your pipeline activities
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# ============================================================
# EXTRACT: Pull data from multiple sources
# ============================================================

def extract_from_csv(filepath):
    """Extract data from a CSV file."""
    logger.info(f"Extracting data from {filepath}")
    try:
        df = pd.read_csv(filepath)
        logger.info(f"Extracted {len(df)} rows from {filepath}")
        return df
    except FileNotFoundError:
        logger.error(f"File not found: {filepath}")
        raise

def extract_from_api(url, params=None):
    """Extract data from a REST API."""
    import requests
    
    logger.info(f"Extracting data from API: {url}")
    response = requests.get(url, params=params, timeout=30)
    response.raise_for_status()  # Raises exception for 4XX/5XX status codes
    
    data = response.json()
    df = pd.DataFrame(data)
    logger.info(f"Extracted {len(df)} records from API")
    return df

def extract_from_database(connection_string, query):
    """Extract data from a SQL database."""
    logger.info(f"Extracting data from database")
    conn = sqlite3.connect(connection_string)
    df = pd.read_sql_query(query, conn)
    conn.close()
    logger.info(f"Extracted {len(df)} rows from database")
    return df

# ============================================================
# TRANSFORM: Clean and reshape the data
# ============================================================

def transform_sales_data(df):
    """Apply business transformations to sales data."""
    logger.info("Starting transformations...")
    
    # 1. Remove duplicates
    initial_rows = len(df)
    df = df.drop_duplicates(subset=['order_id'])
    logger.info(f"Removed {initial_rows - len(df)} duplicate rows")
    
    # 2. Handle missing values
    # For numerical columns: fill with median (robust to outliers)
    numeric_cols = df.select_dtypes(include=['number']).columns
    for col in numeric_cols:
        null_count = df[col].isnull().sum()
        if null_count > 0:
            median_val = df[col].median()
            df[col] = df[col].fillna(median_val)
            logger.info(f"Filled {null_count} nulls in '{col}' with median={median_val}")
    
    # For categorical columns: fill with mode or 'Unknown'
    cat_cols = df.select_dtypes(include=['object']).columns
    for col in cat_cols:
        df[col] = df[col].fillna('Unknown')
    
    # 3. Standardize formats
    if 'date' in df.columns:
        df['date'] = pd.to_datetime(df['date'], errors='coerce')
        df['year'] = df['date'].dt.year
        df['month'] = df['date'].dt.month
        df['day_of_week'] = df['date'].dt.day_name()
    
    # 4. Business logic: Calculate derived metrics
    if 'quantity' in df.columns and 'unit_price' in df.columns:
        df['total_revenue'] = df['quantity'] * df['unit_price']
        df['revenue_category'] = pd.cut(
            df['total_revenue'],
            bins=[0, 50, 200, 1000, float('inf')],
            labels=['Small', 'Medium', 'Large', 'Enterprise']
        )
    
    # 5. Data validation
    # Remove rows with negative quantities (data error)
    if 'quantity' in df.columns:
        invalid_rows = (df['quantity'] < 0).sum()
        if invalid_rows > 0:
            logger.warning(f"Removing {invalid_rows} rows with negative quantity")
            df = df[df['quantity'] >= 0]
    
    logger.info(f"Transformation complete. Final shape: {df.shape}")
    return df

# ============================================================
# LOAD: Store processed data
# ============================================================

def load_to_database(df, table_name, db_path, if_exists='append'):
    """Load DataFrame into a SQLite database."""
    logger.info(f"Loading {len(df)} rows into '{table_name}'")
    conn = sqlite3.connect(db_path)
    df.to_sql(table_name, conn, if_exists=if_exists, index=False)
    conn.close()
    logger.info(f"Successfully loaded data into '{table_name}'")

def load_to_parquet(df, filepath):
    """Load DataFrame as a Parquet file (columnar, compressed)."""
    logger.info(f"Saving {len(df)} rows to {filepath}")
    df.to_parquet(filepath, index=False, compression='snappy')
    logger.info(f"Saved to {filepath}")

# ============================================================
# PIPELINE: Orchestrate the full ETL
# ============================================================

def run_sales_pipeline():
    """Main pipeline function — orchestrates Extract, Transform, Load."""
    pipeline_start = datetime.now()
    logger.info("=" * 50)
    logger.info("STARTING SALES ETL PIPELINE")
    logger.info("=" * 50)
    
    try:
        # EXTRACT
        sales_df = extract_from_csv('data/raw/sales_2024.csv')
        
        # TRANSFORM
        clean_df = transform_sales_data(sales_df)
        
        # LOAD
        load_to_database(clean_df, 'fact_sales', 'data/warehouse.db', if_exists='replace')
        load_to_parquet(clean_df, 'data/processed/sales_clean.parquet')
        
        # Pipeline metrics
        duration = (datetime.now() - pipeline_start).total_seconds()
        logger.info(f"Pipeline completed in {duration:.2f} seconds")
        logger.info(f"Rows processed: {len(clean_df)}")
        
        return {"status": "success", "rows": len(clean_df), "duration": duration}
    
    except Exception as e:
        logger.error(f"Pipeline FAILED: {str(e)}")
        # In production: send alert (email, Slack, PagerDuty)
        return {"status": "failed", "error": str(e)}

# Run the pipeline
if __name__ == "__main__":
    result = run_sales_pipeline()
    print(result)
```

### Incremental Loading (Don't Re-process Everything)

```python
import pandas as pd
import sqlite3
from datetime import datetime, timedelta

def get_last_processed_timestamp(db_path, table_name):
    """Get the timestamp of the last successfully loaded record."""
    conn = sqlite3.connect(db_path)
    query = f"SELECT MAX(loaded_at) FROM {table_name}"
    result = pd.read_sql_query(query, conn)
    conn.close()
    
    last_ts = result.iloc[0, 0]
    if last_ts is None:
        # First run — get everything from 30 days ago
        return (datetime.now() - timedelta(days=30)).isoformat()
    return last_ts

def extract_incremental(source_db, last_timestamp):
    """Only extract records newer than last processed timestamp."""
    conn = sqlite3.connect(source_db)
    query = f"""
        SELECT * FROM orders 
        WHERE created_at > '{last_timestamp}'
        ORDER BY created_at ASC
    """
    df = pd.read_sql_query(query, conn)
    conn.close()
    print(f"Extracted {len(df)} new records since {last_timestamp}")
    return df

def load_with_timestamp(df, db_path, table_name):
    """Load data with a 'loaded_at' timestamp for tracking."""
    df = df.copy()
    df['loaded_at'] = datetime.now().isoformat()
    
    conn = sqlite3.connect(db_path)
    df.to_sql(table_name, conn, if_exists='append', index=False)
    conn.close()

# Usage: Incremental pipeline
last_ts = get_last_processed_timestamp('warehouse.db', 'fact_orders')
new_data = extract_incremental('source.db', last_ts)

if len(new_data) > 0:
    transformed = transform_sales_data(new_data)
    load_with_timestamp(transformed, 'warehouse.db', 'fact_orders')
else:
    print("No new data to process")
```

> **Pro Tip:** Always prefer incremental loads over full loads. Re-processing millions of rows daily when only thousands changed is wasteful and slow. Use watermarks (timestamps or IDs) to track what's already been processed.

---

## Apache Airflow — Orchestrating Pipelines

### What is Airflow?

Airflow is the most popular open-source tool for **scheduling and monitoring** data pipelines. Think of it as a "cron job on steroids" — it handles dependencies, retries, logging, and visualization of your workflows.

### Core Concepts

| Concept | What It Is | Analogy |
|---------|-----------|---------|
| **DAG** | Directed Acyclic Graph — defines pipeline structure | Recipe with steps |
| **Task** | A single unit of work | One step in the recipe |
| **Operator** | Template for a task type | Type of cooking action (chop, boil, fry) |
| **Dependency** | Order between tasks | "You must chop before you fry" |
| **Schedule** | When the DAG runs | "Every day at 6 AM" |
| **XCom** | Data passed between tasks | Passing ingredients between cooks |

### DAG Example

```python
# dags/sales_etl_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.sensors.filesystem import FileSensor
from datetime import datetime, timedelta

# Default arguments applied to all tasks in this DAG
default_args = {
    'owner': 'data_team',
    'depends_on_past': False,             # Don't wait for previous day's run
    'email': ['data-alerts@company.com'],
    'email_on_failure': True,             # Send email if task fails
    'email_on_retry': False,
    'retries': 3,                          # Retry failed tasks 3 times
    'retry_delay': timedelta(minutes=5),   # Wait 5 min between retries
}

# Define the DAG
dag = DAG(
    dag_id='sales_etl_pipeline',
    default_args=default_args,
    description='Daily sales data ETL pipeline',
    schedule_interval='0 6 * * *',  # Every day at 6:00 AM UTC
    start_date=datetime(2024, 1, 1),
    catchup=False,  # Don't backfill missed runs
    tags=['sales', 'etl', 'production'],
)

# ── Task Functions ──────────────────────────────────────────

def extract_sales(**context):
    """Extract sales data from source systems."""
    import pandas as pd
    
    # execution_date is the logical date of this DAG run
    exec_date = context['ds']  # Format: 'YYYY-MM-DD'
    print(f"Extracting sales data for {exec_date}")
    
    df = pd.read_csv(f'/data/raw/sales_{exec_date}.csv')
    # Save to intermediate location
    df.to_parquet(f'/data/staging/sales_{exec_date}.parquet')
    
    # Return metadata via XCom
    return {'rows_extracted': len(df), 'date': exec_date}

def transform_sales(**context):
    """Transform and clean sales data."""
    import pandas as pd
    
    exec_date = context['ds']
    df = pd.read_parquet(f'/data/staging/sales_{exec_date}.parquet')
    
    # Apply transformations
    df = df.drop_duplicates()
    df['revenue'] = df['quantity'] * df['price']
    df['date'] = pd.to_datetime(df['date'])
    
    # Save transformed data
    df.to_parquet(f'/data/processed/sales_{exec_date}.parquet')
    return {'rows_transformed': len(df)}

def load_to_warehouse(**context):
    """Load processed data into the data warehouse."""
    import pandas as pd
    from sqlalchemy import create_engine
    
    exec_date = context['ds']
    df = pd.read_parquet(f'/data/processed/sales_{exec_date}.parquet')
    
    engine = create_engine('postgresql://user:pass@warehouse:5432/analytics')
    df.to_sql('fact_sales', engine, if_exists='append', index=False)
    print(f"Loaded {len(df)} rows into fact_sales")

def run_data_quality_checks(**context):
    """Validate data quality after loading."""
    from sqlalchemy import create_engine
    import pandas as pd
    
    exec_date = context['ds']
    engine = create_engine('postgresql://user:pass@warehouse:5432/analytics')
    
    # Check 1: No null revenues
    result = pd.read_sql(
        f"SELECT COUNT(*) as nulls FROM fact_sales WHERE revenue IS NULL AND date = '{exec_date}'",
        engine
    )
    assert result['nulls'][0] == 0, "Found null revenues!"
    
    # Check 2: Revenue should be positive
    result = pd.read_sql(
        f"SELECT COUNT(*) as negatives FROM fact_sales WHERE revenue < 0 AND date = '{exec_date}'",
        engine
    )
    assert result['negatives'][0] == 0, "Found negative revenues!"
    
    print("All data quality checks passed!")

# ── Define Tasks ────────────────────────────────────────────

# Task 1: Wait for source file to appear
wait_for_file = FileSensor(
    task_id='wait_for_source_file',
    filepath='/data/raw/sales_{{ ds }}.csv',
    poke_interval=300,  # Check every 5 minutes
    timeout=3600,       # Give up after 1 hour
    dag=dag,
)

# Task 2: Extract
extract_task = PythonOperator(
    task_id='extract_sales_data',
    python_callable=extract_sales,
    dag=dag,
)

# Task 3: Transform
transform_task = PythonOperator(
    task_id='transform_sales_data',
    python_callable=transform_sales,
    dag=dag,
)

# Task 4: Load
load_task = PythonOperator(
    task_id='load_to_warehouse',
    python_callable=load_to_warehouse,
    dag=dag,
)

# Task 5: Quality checks
quality_task = PythonOperator(
    task_id='data_quality_checks',
    python_callable=run_data_quality_checks,
    dag=dag,
)

# Task 6: Notify success
notify_task = BashOperator(
    task_id='notify_success',
    bash_command='echo "Pipeline completed successfully for {{ ds }}"',
    dag=dag,
)

# ── Define Dependencies ─────────────────────────────────────
# This creates the DAG structure:
# wait_for_file → extract → transform → load → quality_checks → notify

wait_for_file >> extract_task >> transform_task >> load_task >> quality_task >> notify_task
```

### Airflow DAG Visualization

```
┌─────────────────┐     ┌───────────────┐     ┌─────────────────┐
│ wait_for_file   │────▶│ extract_sales │────▶│ transform_sales │
└─────────────────┘     └───────────────┘     └────────┬────────┘
                                                        │
                                                        ▼
┌─────────────────┐     ┌───────────────────┐   ┌──────┴────────┐
│ notify_success  │◀────│ data_quality_check│◀──│load_to_warehouse│
└─────────────────┘     └───────────────────┘   └───────────────┘
```

---

## Other Pipeline Tools

### Tool Comparison

| Tool | Type | Best For | Learning Curve | Scale |
|------|------|----------|----------------|-------|
| **Apache Airflow** | Orchestrator | Complex DAGs, large teams | Steep | Enterprise |
| **Prefect** | Orchestrator | Python-native, modern UX | Moderate | Mid-Large |
| **Dagster** | Orchestrator | Data-asset focused, testing | Moderate | Mid-Large |
| **Luigi** | Orchestrator | Simple pipelines | Easy | Small-Mid |
| **dbt** | Transform only | SQL transformations in warehouse | Easy | Any |
| **Apache Spark** | Processing engine | Big data (TB+) | Steep | Enterprise |
| **Apache Kafka** | Streaming | Real-time event processing | Steep | Enterprise |
| **Fivetran/Airbyte** | Extract+Load | Automated connectors (no code) | Easy | Any |

### Prefect — Modern Python-Native Orchestration

```python
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta
import pandas as pd

# Tasks are decorated functions with built-in retry/caching
@task(retries=3, retry_delay_seconds=60, cache_key_fn=task_input_hash, cache_expiration=timedelta(hours=1))
def extract(source_path: str) -> pd.DataFrame:
    """Extract data — cached for 1 hour to avoid redundant reads."""
    print(f"Reading from {source_path}")
    return pd.read_csv(source_path)

@task
def transform(df: pd.DataFrame) -> pd.DataFrame:
    """Clean and transform data."""
    df = df.drop_duplicates()
    df['total'] = df['quantity'] * df['price']
    return df

@task
def load(df: pd.DataFrame, dest_path: str):
    """Load to destination."""
    df.to_parquet(dest_path, index=False)
    print(f"Saved {len(df)} rows to {dest_path}")

# Flow is the pipeline definition
@flow(name="sales-etl", log_prints=True)
def sales_pipeline(source: str, destination: str):
    """Main pipeline flow."""
    raw_data = extract(source)
    clean_data = transform(raw_data)
    load(clean_data, destination)

# Run it
if __name__ == "__main__":
    sales_pipeline(
        source="data/sales.csv",
        destination="data/output/sales_clean.parquet"
    )
```

### dbt — SQL-Based Transformations

```sql
-- models/staging/stg_orders.sql
-- dbt model: cleans raw orders into staging table

WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
),

cleaned AS (
    SELECT
        order_id,
        customer_id,
        CAST(order_date AS DATE) AS order_date,
        COALESCE(status, 'unknown') AS status,
        ROUND(amount, 2) AS amount,
        -- Remove test orders
        CASE WHEN customer_id LIKE 'TEST%' THEN TRUE ELSE FALSE END AS is_test
    FROM source
    WHERE order_date IS NOT NULL  -- Remove records with no date
)

SELECT * FROM cleaned WHERE NOT is_test
```

```yaml
# models/staging/schema.yml
# dbt schema file — defines tests and documentation

version: 2

models:
  - name: stg_orders
    description: "Cleaned orders from raw source"
    columns:
      - name: order_id
        description: "Unique order identifier"
        tests:
          - unique        # Fail if duplicates exist
          - not_null      # Fail if nulls exist
      - name: amount
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0  # Amount should never be negative
```

---

## Data Quality and Validation

### Why Data Quality Matters

> "Garbage in, garbage out." — If your pipeline loads bad data, every dashboard and ML model downstream is wrong. Data quality is not optional.

### Great Expectations — Data Validation Framework

```python
import great_expectations as gx
import pandas as pd

# Create a data context
context = gx.get_context()

# Load your data
df = pd.read_csv("data/processed/sales.csv")

# Define expectations (rules your data must follow)
validator = context.sources.pandas_default.read_dataframe(df)

# Column should never be null
validator.expect_column_values_to_not_be_null("order_id")

# Revenue should be between 0 and 1,000,000
validator.expect_column_values_to_be_between(
    "revenue", min_value=0, max_value=1_000_000
)

# Date should be in valid format
validator.expect_column_values_to_match_regex(
    "date", regex=r"\d{4}-\d{2}-\d{2}"
)

# No more than 5% nulls in any column
validator.expect_column_values_to_not_be_null(
    "customer_name", mostly=0.95  # Allow up to 5% nulls
)

# Table should have at least 1000 rows (detect truncated loads)
validator.expect_table_row_count_to_be_between(
    min_value=1000, max_value=10_000_000
)

# Run validation
results = validator.validate()
print(f"Validation {'PASSED' if results.success else 'FAILED'}")
```

### Custom Data Quality Checks

```python
import pandas as pd
from dataclasses import dataclass
from typing import List, Callable

@dataclass
class QualityCheck:
    name: str
    check_fn: Callable[[pd.DataFrame], bool]
    severity: str  # 'critical' or 'warning'

def run_quality_checks(df: pd.DataFrame, checks: List[QualityCheck]) -> dict:
    """Run all quality checks and return results."""
    results = {"passed": [], "failed": [], "warnings": []}
    
    for check in checks:
        try:
            passed = check.check_fn(df)
            if passed:
                results["passed"].append(check.name)
            elif check.severity == 'critical':
                results["failed"].append(check.name)
            else:
                results["warnings"].append(check.name)
        except Exception as e:
            results["failed"].append(f"{check.name}: {str(e)}")
    
    return results

# Define checks
checks = [
    QualityCheck(
        name="no_duplicate_orders",
        check_fn=lambda df: df['order_id'].is_unique,
        severity="critical"
    ),
    QualityCheck(
        name="revenue_is_positive",
        check_fn=lambda df: (df['revenue'] >= 0).all(),
        severity="critical"
    ),
    QualityCheck(
        name="no_future_dates",
        check_fn=lambda df: (pd.to_datetime(df['date']) <= pd.Timestamp.now()).all(),
        severity="warning"
    ),
    QualityCheck(
        name="row_count_reasonable",
        check_fn=lambda df: 100 <= len(df) <= 10_000_000,
        severity="critical"
    ),
    QualityCheck(
        name="null_rate_below_5pct",
        check_fn=lambda df: (df.isnull().sum() / len(df)).max() < 0.05,
        severity="warning"
    ),
]

# Run
df = pd.read_csv("data/processed/sales.csv")
results = run_quality_checks(df, checks)

print(f"✓ Passed: {len(results['passed'])}")
print(f"✗ Failed: {len(results['failed'])}")
print(f"⚠ Warnings: {len(results['warnings'])}")

# In production: block pipeline if critical checks fail
if results['failed']:
    raise RuntimeError(f"Critical quality checks failed: {results['failed']}")
```

---

## Streaming vs Batch Processing

### The Difference

| Aspect | Batch Processing | Stream Processing |
|--------|-----------------|-------------------|
| Data arrival | Collected over time, processed together | Processed as it arrives |
| Latency | Minutes to hours | Milliseconds to seconds |
| Use case | Reports, ML training, aggregations | Fraud detection, real-time dashboards |
| Complexity | Simpler | More complex (ordering, late data) |
| Tools | Spark, Pandas, SQL | Kafka, Flink, Spark Streaming |
| Cost | Lower (scheduled) | Higher (always running) |

### When to Use Each

```
Use BATCH when:
├── Data freshness of hours/days is acceptable
├── You're doing complex aggregations/joins
├── Cost efficiency is important
└── Example: "Daily sales summary", "Monthly churn report"

Use STREAMING when:
├── You need sub-second latency
├── Data is naturally event-driven
├── Delayed processing has business cost
└── Example: "Fraud alert", "Real-time pricing", "Live dashboard"
```

### Simple Streaming Example with Kafka + Python

```python
# Producer: Sends events to Kafka topic
from kafka import KafkaProducer
import json
from datetime import datetime

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Simulate sending a transaction event
event = {
    "transaction_id": "txn_12345",
    "user_id": "user_789",
    "amount": 299.99,
    "merchant": "electronics_store",
    "timestamp": datetime.now().isoformat()
}

producer.send('transactions', value=event)
producer.flush()
print(f"Sent event: {event['transaction_id']}")
```

```python
# Consumer: Processes events from Kafka in real-time
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'transactions',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    auto_offset_reset='latest',  # Only read new messages
    group_id='fraud_detection_group'
)

def is_suspicious(transaction):
    """Simple fraud detection rules."""
    if transaction['amount'] > 5000:
        return True, "High amount"
    # Add more rules: unusual location, frequency, etc.
    return False, None

# Process events as they arrive (infinite loop)
print("Listening for transactions...")
for message in consumer:
    txn = message.value
    suspicious, reason = is_suspicious(txn)
    
    if suspicious:
        print(f"🚨 ALERT: Suspicious transaction {txn['transaction_id']}")
        print(f"   Reason: {reason}, Amount: ${txn['amount']}")
        # In production: trigger alert, block card, notify user
    else:
        print(f"✓ Normal transaction: {txn['transaction_id']}")
```

---

## Common Mistakes

### 1. No Idempotency
```python
# ❌ BAD: Running pipeline twice creates duplicate data
def load(df, table):
    df.to_sql(table, engine, if_exists='append')  # Appends every run!

# ✓ GOOD: Pipeline can be safely re-run
def load(df, table, date):
    # Delete existing data for this date, then insert
    engine.execute(f"DELETE FROM {table} WHERE date = '{date}'")
    df.to_sql(table, engine, if_exists='append')
```

### 2. No Error Handling or Alerting
```python
# ❌ BAD: Silent failures
def pipeline():
    df = extract()
    df = transform(df)
    load(df)

# ✓ GOOD: Proper error handling with alerts
def pipeline():
    try:
        df = extract()
        df = transform(df)
        load(df)
    except Exception as e:
        send_slack_alert(f"Pipeline failed: {e}")
        logger.error(f"Pipeline failed", exc_info=True)
        raise  # Re-raise so orchestrator marks it as failed
```

### 3. Processing Everything Every Time
```python
# ❌ BAD: Full load every day (slow, expensive)
df = pd.read_sql("SELECT * FROM orders", source_engine)  # Millions of rows!

# ✓ GOOD: Incremental load
last_id = get_last_processed_id()
df = pd.read_sql(f"SELECT * FROM orders WHERE id > {last_id}", source_engine)
```

### 4. No Data Quality Checks
```python
# ❌ BAD: Blindly loading whatever comes in
load(transformed_df)

# ✓ GOOD: Validate before loading
assert len(transformed_df) > 0, "Empty dataframe — source may be down"
assert transformed_df['id'].is_unique, "Duplicate IDs detected"
assert (transformed_df['amount'] >= 0).all(), "Negative amounts found"
load(transformed_df)
```

### 5. Hardcoded Configuration
```python
# ❌ BAD: Hardcoded paths and credentials
conn = sqlite3.connect('/home/john/data/prod.db')
API_KEY = "sk-abc123xyz"

# ✓ GOOD: Use environment variables and config files
import os
conn = sqlite3.connect(os.environ['DATABASE_PATH'])
API_KEY = os.environ['API_KEY']
```

---

## Interview Questions

### Conceptual Questions

**Q1: What's the difference between ETL and ELT? When would you choose one over the other?**

> **Answer:** ETL transforms data before loading into the warehouse. ELT loads raw data first, then transforms inside the warehouse. Choose ETL when you have compliance requirements (sensitive data must be filtered before storage) or limited warehouse compute. Choose ELT when using modern cloud warehouses (Snowflake, BigQuery) where compute is elastic and you want flexibility to re-transform data without re-extracting.

**Q2: How would you make a pipeline idempotent?**

> **Answer:** An idempotent pipeline produces the same result whether run once or multiple times. Strategies:
> 1. Use "DELETE + INSERT" instead of just "INSERT" (partition-based overwrites)
> 2. Use MERGE/UPSERT operations with unique keys
> 3. Write to date-partitioned tables and overwrite the partition
> 4. Use deduplication logic with unique identifiers

**Q3: How do you handle late-arriving data in a streaming pipeline?**

> **Answer:** Use watermarks — a threshold that defines how late data can arrive. Data arriving after the watermark is either discarded or sent to a "late data" side output for separate processing. In Spark Streaming, you'd use `withWatermark("event_time", "10 minutes")`. In Flink, you'd configure allowed lateness in window operations.

**Q4: Your daily pipeline suddenly produces 50% fewer rows than usual. What do you do?**

> **Answer:** 
> 1. Check if the source system had an outage or partial data
> 2. Check if filtering logic accidentally removed valid rows
> 3. Compare schema — did a column rename break a join?
> 4. Check for date range issues (timezone bugs are common)
> 5. Look at pipeline logs for errors or warnings
> 6. Have row count anomaly detection that alerts automatically

**Q5: How would you design a pipeline that needs to process 10TB of data daily?**

> **Answer:** 
> - Use distributed processing (Spark/Dask, not Pandas)
> - Partition data by date/region for parallel processing
> - Use columnar formats (Parquet) for efficient reads
> - Implement incremental processing
> - Use a cluster (EMR, Dataproc, Databricks)
> - Consider whether all 10TB needs to be processed or just the delta

### System Design Questions

**Q6: Design a real-time recommendation pipeline for an e-commerce site.**

> **Answer:**
> ```
> User clickstream → Kafka → Feature computation (Flink) → Feature Store
>                                                               ↓
> User request → API → Fetch features → ML Model → Ranked recommendations
>                                                               ↓
>                                                     Log to data lake
>                                                     (for model retraining)
> ```
> Key considerations: latency (<100ms), feature freshness, model serving, A/B testing integration.

---

## Quick Reference

### ETL Pipeline Checklist

| Step | Action | Tool |
|------|--------|------|
| 1 | Define data sources and destinations | Documentation |
| 2 | Build extraction logic | Python, Fivetran, Airbyte |
| 3 | Design transformations | Pandas, Spark, dbt |
| 4 | Implement data quality checks | Great Expectations, custom |
| 5 | Set up orchestration/scheduling | Airflow, Prefect, Dagster |
| 6 | Add monitoring and alerting | Datadog, PagerDuty, Slack |
| 7 | Document the pipeline | README, lineage tools |
| 8 | Test with sample data | pytest, dbt tests |

### Key Commands

```bash
# Airflow
airflow dags list                # List all DAGs
airflow dags trigger my_dag      # Manually trigger a DAG
airflow tasks test my_dag my_task 2024-01-01  # Test single task

# dbt
dbt run                          # Run all models
dbt test                         # Run all tests
dbt run --select staging.*       # Run only staging models

# Prefect
prefect deployment run my-flow/my-deployment  # Trigger a flow
```

### Pipeline Design Principles

| Principle | Description |
|-----------|-------------|
| **Idempotent** | Same result if run once or 10 times |
| **Atomic** | Either fully succeeds or fully fails (no partial loads) |
| **Incremental** | Only process new/changed data |
| **Observable** | Logs, metrics, alerts at every step |
| **Testable** | Unit tests for transform logic, integration tests for full pipeline |
| **Documented** | Clear lineage: where data comes from and goes |
| **Versioned** | Pipeline code in git, schema migrations tracked |

---

*Next: [Chapter 08 - SQL for Data Science](08-SQL-for-Data-Science.md)*
