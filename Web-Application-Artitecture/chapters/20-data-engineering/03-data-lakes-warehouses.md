# Data Lakes & Data Warehouses (S3, BigQuery, Snowflake, Redshift)

> **What you'll learn**: The difference between dumping all your raw data into a "lake" vs organizing it into a structured "warehouse" — when to use each, how they work under the hood, and why most companies end up using both (the "Lakehouse" pattern).

---

## Real-Life Analogy

Imagine you're archiving **every piece of mail your company ever received**.

### The Data Lake (A Giant Storage Room)

You have an enormous warehouse building where you throw **everything** — unopened mail, packages, faxes, emails printed out, sticky notes, napkin sketches. Nothing is organized. Everything is kept "just in case."

- **Cheap storage** — just shelves and boxes
- **No structure required** — toss anything in
- **Finding anything specific?** You'd need to dig through boxes
- **But it's all there** — nothing was thrown away

### The Data Warehouse (A Well-Organized Library)

You also have a **library** next door. Here, every document is:
- Catalogued with a card system (schema)
- Organized by topic (tables)
- Cross-referenced (joins)
- Ready to answer questions immediately

- **More expensive** — librarians needed to organize
- **Strict structure** — must fit the catalogue system
- **Fast answers** — "How many contracts did we sign last quarter?" → instant
- **But some raw data was discarded** — only "important" stuff made it here

### The Modern Truth

Today's companies need **both** — the raw archive AND the organized library. That's the **Lakehouse** pattern.

---

## Core Concept Explained Step-by-Step

### Step 1: What is a Data Lake?

A **data lake** is a centralized repository that stores **all** your data in its **raw, native format** — structured (tables), semi-structured (JSON, XML), and unstructured (images, logs, videos).

```
┌─────────────────────────────────────────────────────────────────┐
│                         DATA LAKE                                 │
│                    (e.g., Amazon S3, GCS)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  s3://company-data-lake/                                          │
│  ├── raw/                                                         │
│  │   ├── orders/2024/06/15/orders_20240615.parquet               │
│  │   ├── clickstream/2024/06/15/clicks_00.json.gz                │
│  │   ├── server-logs/2024/06/app-server-1.log                    │
│  │   └── images/products/SKU12345.jpg                            │
│  ├── processed/                                                   │
│  │   ├── orders_cleaned/2024/06/15/*.parquet                     │
│  │   └── user_features/2024/06/15/*.parquet                      │
│  └── curated/                                                     │
│      └── ml_training_data/model_v3/*.parquet                     │
│                                                                   │
│  Storage: $0.023/GB/month (S3 Standard)                           │
│  Format: Parquet, ORC, JSON, Avro, CSV, images, video             │
│  Compute: Separate (Spark, Presto, Athena query the files)        │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: What is a Data Warehouse?

A **data warehouse** is a purpose-built analytical database that stores **structured, processed data** in a format optimized for fast SQL queries.

```
┌─────────────────────────────────────────────────────────────────┐
│                       DATA WAREHOUSE                              │
│                  (e.g., BigQuery, Snowflake)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Schema: analytics                                                │
│  ├── fact_orders          (2.5 billion rows, columnar)            │
│  ├── fact_page_views      (50 billion rows, columnar)             │
│  ├── dim_users            (50 million rows)                       │
│  ├── dim_products         (2 million rows)                        │
│  ├── dim_dates            (10,000 rows)                           │
│  └── mart_monthly_revenue (aggregated summary)                    │
│                                                                   │
│  Storage: ~$25-40/TB/month (compressed columnar)                  │
│  Format: Proprietary columnar (optimized for SQL)                 │
│  Compute: Built-in, auto-scaling query engine                     │
│  Schema: REQUIRED (must define columns, types)                    │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: Key Differences

```
┌────────────────┬─────────────────────────────┬─────────────────────────────┐
│ Characteristic │       DATA LAKE              │       DATA WAREHOUSE         │
├────────────────┼─────────────────────────────┼─────────────────────────────┤
│ Data Format    │ Raw, any format             │ Structured, columnar         │
│ Schema         │ Schema-on-Read              │ Schema-on-Write              │
│ Users          │ Data engineers, ML engineers │ Analysts, BI tools           │
│ Cost           │ Very cheap ($0.02/GB/mo)    │ More expensive ($25+/TB/mo)  │
│ Query Speed    │ Slow (scan raw files)       │ Fast (optimized engine)      │
│ Data Types     │ Structured + unstructured   │ Structured only              │
│ Processing     │ Separate compute (Spark)    │ Built-in compute             │
│ Governance     │ Can become "data swamp"     │ Enforced schemas & access    │
│ Use Cases      │ ML training, archival, raw  │ BI, dashboards, reporting    │
│ Updates        │ Difficult (immutable files) │ Supported (MERGE/UPSERT)     │
└────────────────┴─────────────────────────────┴─────────────────────────────┘
```

### Step 4: The Lakehouse — Best of Both Worlds

Modern architectures combine both into a **Lakehouse**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LAKEHOUSE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │              Query Engine (SQL Interface)                       │      │
│  │         Spark SQL / Trino / Athena / Databricks SQL            │      │
│  └───────────────────────┬───────────────────────────────────────┘      │
│                          │                                               │
│  ┌───────────────────────┴───────────────────────────────────────┐      │
│  │            Table Format Layer (Structure on Lake)               │      │
│  │         Delta Lake / Apache Iceberg / Apache Hudi              │      │
│  │                                                                │      │
│  │  Provides: ACID transactions, schema enforcement, time travel, │      │
│  │           partition pruning, metadata management                │      │
│  └───────────────────────┬───────────────────────────────────────┘      │
│                          │                                               │
│  ┌───────────────────────┴───────────────────────────────────────┐      │
│  │              Object Storage (Data Lake)                         │      │
│  │         Amazon S3 / Google Cloud Storage / Azure ADLS          │      │
│  │                                                                │      │
│  │  Stores: Parquet/ORC files, cheap, infinite scale              │      │
│  └───────────────────────────────────────────────────────────────┘      │
│                                                                          │
│  Result: Lake's cost & flexibility + Warehouse's speed & governance      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### BigQuery Architecture (Google)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       BIGQUERY INTERNALS                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  User submits SQL query                                                  │
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────────┐                                                     │
│  │  Dremel Engine   │  ← Query coordinator                               │
│  │  (Root Server)   │    Parses SQL, creates execution plan               │
│  └────────┬────────┘                                                     │
│           │                                                              │
│           │ Distributes work to thousands of leaf nodes                   │
│           │                                                              │
│     ┌─────┴─────┬─────────────┬─────────────┐                           │
│     ▼           ▼             ▼             ▼                            │
│  ┌──────┐   ┌──────┐     ┌──────┐     ┌──────┐                          │
│  │Leaf 1│   │Leaf 2│     │Leaf 3│     │Leaf N│  ← Thousands of workers   │
│  └──┬───┘   └──┬───┘     └──┬───┘     └──┬───┘    scan in parallel       │
│     │           │             │             │                             │
│     ▼           ▼             ▼             ▼                             │
│  ┌──────────────────────────────────────────────┐                        │
│  │            Colossus (Storage Layer)            │                        │
│  │                                               │                        │
│  │  Data stored in Capacitor format (columnar)   │                        │
│  │  Replicated across data centers               │                        │
│  │  Petabytes of data                            │                        │
│  └──────────────────────────────────────────────┘                        │
│                                                                          │
│  Key Innovation: Storage & Compute are SEPARATED                         │
│  → Scale compute independently                                           │
│  → Pay only for queries, not idle servers                                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Snowflake Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       SNOWFLAKE ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Layer 1: CLOUD SERVICES (Brain)                                         │
│  ┌───────────────────────────────────────────────────┐                   │
│  │  Authentication, Query Parsing, Optimization,      │                   │
│  │  Metadata, Access Control, Infrastructure Mgmt     │                   │
│  └───────────────────────────┬───────────────────────┘                   │
│                              │                                           │
│  Layer 2: COMPUTE (Virtual Warehouses)                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                               │
│  │  XS WH    │  │  L WH     │  │  XL WH    │  ← Independent clusters    │
│  │  (Analyst │  │  (BI Tool │  │  (Data     │    Can scale up/down        │
│  │   queries)│  │   queries)│  │   science) │    independently            │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘                               │
│        │              │              │                                    │
│  Layer 3: STORAGE (Shared)                                               │
│  ┌───────────────────────────────────────────────────┐                   │
│  │  S3 / Azure Blob / GCS                             │                   │
│  │  Micro-partitions (50-500MB compressed columnar)   │                   │
│  │  Immutable, automatically clustered                │                   │
│  └───────────────────────────────────────────────────┘                   │
│                                                                          │
│  Key Innovation: Multiple compute clusters share ONE storage             │
│  → No data copying between teams                                         │
│  → Each team gets own performance isolation                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### Amazon Redshift Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      REDSHIFT ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────┐                                                         │
│  │ Leader Node  │ ← Receives queries, creates execution plan             │
│  └──────┬──────┘                                                         │
│         │ Distributes to compute nodes                                   │
│    ┌────┼────┬────────┐                                                  │
│    ▼    ▼    ▼        ▼                                                  │
│  ┌────┐┌────┐┌────┐┌────┐                                               │
│  │ C1  ││ C2  ││ C3  ││ C4  │ ← Compute Nodes                            │
│  │    ││    ││    ││    │   (each stores a portion of data)             │
│  │ S1 ││ S1 ││ S1 ││ S1 │   S = Slice (local columnar storage)          │
│  │ S2 ││ S2 ││ S2 ││ S2 │                                               │
│  └────┘└────┘└────┘└────┘                                               │
│                                                                          │
│  Distribution Styles:                                                    │
│  • KEY: Rows with same key → same node (good for JOINs)                 │
│  • ALL: Full table copy on every node (small dimension tables)           │
│  • EVEN: Round-robin across nodes (default)                              │
│                                                                          │
│  Also supports: Redshift Spectrum (query S3 data lake directly)          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Writing to Data Lake (S3 + Parquet)

```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import boto3
from datetime import date

# ═══════════════════════════════════════════════════════
# Writing data to a Data Lake in Parquet format
# Partitioned by date for efficient querying
# ═══════════════════════════════════════════════════════

def write_to_data_lake(orders_df: pd.DataFrame, execution_date: date):
    """
    Write order data to S3 data lake in Parquet format.
    Partitioned by year/month/day for partition pruning.
    """
    # Convert to PyArrow Table for efficient Parquet writing
    table = pa.Table.from_pandas(orders_df)
    
    # Write to S3 with date-based partitioning
    s3_path = (
        f"s3://company-data-lake/raw/orders/"
        f"year={execution_date.year}/"
        f"month={execution_date.month:02d}/"
        f"day={execution_date.day:02d}/"
        f"orders.parquet"
    )
    
    pq.write_table(
        table,
        s3_path,
        compression='snappy',          # Good compression + fast decompression
        row_group_size=100_000,        # 100K rows per row group
        use_dictionary=True,           # Dictionary encoding for repeated values
    )
    print(f"Written {len(orders_df)} rows to {s3_path}")


def query_data_lake_with_athena():
    """
    Query the data lake using AWS Athena (Presto/Trino engine).
    Athena reads Parquet files directly from S3.
    """
    import boto3
    
    client = boto3.client('athena')
    
    # SQL query runs against files in S3!
    query = """
        SELECT 
            product_category,
            SUM(amount) as total_revenue,
            COUNT(*) as order_count
        FROM data_lake.raw_orders
        WHERE year = 2024 AND month = 6   -- Partition pruning!
        GROUP BY product_category
        ORDER BY total_revenue DESC
        LIMIT 20
    """
    
    response = client.start_query_execution(
        QueryString=query,
        QueryExecutionContext={'Database': 'data_lake'},
        ResultConfiguration={'OutputLocation': 's3://query-results-bucket/'}
    )
    
    return response['QueryExecutionId']
```

### Python: Querying Data Warehouse (BigQuery)

```python
from google.cloud import bigquery

# ═══════════════════════════════════════════════════════
# Querying a Data Warehouse (BigQuery)
# Fast, SQL-native, no file management needed
# ═══════════════════════════════════════════════════════

def analyze_user_cohorts():
    """
    Complex analytical query on BigQuery.
    Scans billions of rows in seconds.
    """
    client = bigquery.Client(project='my-project')
    
    query = """
        WITH user_first_purchase AS (
            SELECT 
                user_id,
                MIN(order_date) AS first_purchase_date,
                DATE_TRUNC(MIN(order_date), MONTH) AS cohort_month
            FROM `my-project.analytics.fact_orders`
            GROUP BY user_id
        ),
        monthly_activity AS (
            SELECT
                ufp.cohort_month,
                DATE_DIFF(
                    DATE_TRUNC(fo.order_date, MONTH),
                    ufp.cohort_month,
                    MONTH
                ) AS months_since_first,
                COUNT(DISTINCT fo.user_id) AS active_users
            FROM `my-project.analytics.fact_orders` fo
            JOIN user_first_purchase ufp ON fo.user_id = ufp.user_id
            GROUP BY cohort_month, months_since_first
        )
        SELECT
            cohort_month,
            months_since_first,
            active_users,
            ROUND(
                active_users * 100.0 / FIRST_VALUE(active_users) 
                OVER (PARTITION BY cohort_month ORDER BY months_since_first),
                2
            ) AS retention_pct
        FROM monthly_activity
        WHERE months_since_first <= 12
        ORDER BY cohort_month, months_since_first
    """
    
    # BigQuery scans TBs in seconds, pay per TB scanned ($5/TB)
    job_config = bigquery.QueryJobConfig(
        use_query_cache=True,  # Cache results for repeated queries
    )
    
    results = client.query(query, job_config=job_config).to_dataframe()
    print(f"Query scanned {results.total_bytes_processed / 1e9:.2f} GB")
    return results
```

### Java: Snowflake JDBC Example

```java
import java.sql.*;
import java.util.Properties;

public class SnowflakeWarehouseExample {
    
    /**
     * Connect to Snowflake and run an analytical query.
     * Snowflake separates storage from compute — you pick your warehouse size.
     */
    public void analyzeRevenueTrends() throws SQLException {
        Properties props = new Properties();
        props.put("user", "analyst");
        props.put("password", System.getenv("SNOWFLAKE_PASSWORD"));
        props.put("account", "mycompany.us-east-1");
        props.put("warehouse", "ANALYTICS_WH_MEDIUM");  // Compute cluster
        props.put("db", "PRODUCTION");
        props.put("schema", "ANALYTICS");
        
        Connection conn = DriverManager.getConnection(
            "jdbc:snowflake://mycompany.us-east-1.snowflakecomputing.com",
            props
        );
        
        // Scale up warehouse for heavy query, then scale back down
        Statement stmt = conn.createStatement();
        stmt.execute("ALTER WAREHOUSE ANALYTICS_WH_MEDIUM SET WAREHOUSE_SIZE = 'XLARGE'");
        
        String query = """
            SELECT 
                d.region,
                DATE_TRUNC('month', f.order_date) AS month,
                SUM(f.revenue) AS total_revenue,
                SUM(f.revenue) - LAG(SUM(f.revenue)) 
                    OVER (PARTITION BY d.region ORDER BY DATE_TRUNC('month', f.order_date))
                    AS mom_growth
            FROM ANALYTICS.FACT_ORDERS f
            JOIN ANALYTICS.DIM_GEOGRAPHY d ON f.geo_id = d.geo_id
            WHERE f.order_date >= DATEADD(year, -2, CURRENT_DATE())
            GROUP BY d.region, DATE_TRUNC('month', f.order_date)
            ORDER BY d.region, month
        """;
        
        ResultSet rs = stmt.executeQuery(query);
        while (rs.next()) {
            System.out.printf("Region: %s | Month: %s | Revenue: $%.2f | Growth: $%.2f%n",
                rs.getString("region"),
                rs.getDate("month"),
                rs.getDouble("total_revenue"),
                rs.getDouble("mom_growth"));
        }
        
        // Scale back down to save costs
        stmt.execute("ALTER WAREHOUSE ANALYTICS_WH_MEDIUM SET WAREHOUSE_SIZE = 'MEDIUM'");
        conn.close();
    }
}
```

---

## Infrastructure Examples

### Terraform: Setting Up a Data Lake + Warehouse

```hcl
# AWS S3 Data Lake + Redshift Warehouse + Glue Catalog

# Data Lake: S3 Bucket with lifecycle policies
resource "aws_s3_bucket" "data_lake" {
  bucket = "company-data-lake-prod"
}

resource "aws_s3_bucket_lifecycle_configuration" "lake_lifecycle" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "archive-old-data"
    status = "Enabled"
    
    transition {
      days          = 90
      storage_class = "STANDARD_IA"   # Cheaper after 90 days
    }
    transition {
      days          = 365
      storage_class = "GLACIER"       # Very cheap after 1 year
    }
  }
}

# Data Warehouse: Redshift Serverless
resource "aws_redshiftserverless_namespace" "analytics" {
  namespace_name = "analytics"
  db_name        = "warehouse"
  admin_username = "admin"
  admin_user_password = var.redshift_password
}

resource "aws_redshiftserverless_workgroup" "analytics" {
  namespace_name = aws_redshiftserverless_namespace.analytics.namespace_name
  workgroup_name = "analytics-workgroup"
  base_capacity  = 32  # RPUs (Redshift Processing Units)
}

# Glue Catalog: Makes S3 data queryable with SQL
resource "aws_glue_catalog_database" "lake_db" {
  name = "data_lake"
}

resource "aws_glue_crawler" "orders_crawler" {
  database_name = aws_glue_catalog_database.lake_db.name
  name          = "orders-crawler"
  role          = aws_iam_role.glue_role.arn
  
  s3_target {
    path = "s3://${aws_s3_bucket.data_lake.bucket}/raw/orders/"
  }
  
  schedule = "cron(0 3 * * ? *)"  # Crawl daily at 3 AM
}
```

---

## Comparison of Major Platforms

| Feature | BigQuery | Snowflake | Redshift | Databricks |
|---------|----------|-----------|----------|------------|
| **Type** | Warehouse | Warehouse | Warehouse | Lakehouse |
| **Storage** | Managed (Capacitor) | Managed (S3/Azure/GCS) | Managed | Delta Lake on S3 |
| **Compute** | Serverless (auto) | Virtual Warehouses | Nodes (RA3) | Clusters/Serverless |
| **Pricing** | Pay per TB scanned | Pay per second of compute | Pay per node-hour | Pay per DBU |
| **Separation** | Full (storage ≠ compute) | Full | Partial (RA3) | Full |
| **Best For** | Ad-hoc analytics, GCP | Multi-cloud, data sharing | AWS-native | ML + Analytics |
| **Semi-structured** | STRUCT, ARRAY, JSON | VARIANT type | SUPER type | Native JSON |
| **Streaming** | BigQuery Streaming | Snowpipe | Kinesis Firehose | Structured Streaming |
| **ML Built-in** | BigQuery ML | Snowpark | Redshift ML | MLflow + Spark |
| **Cost (1TB query)** | ~$5 | Varies by WH size | Varies by node | Varies by cluster |

---

## Real-World Example

### Netflix: Lake + Warehouse Hybrid

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NETFLIX DATA ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Sources (Petabytes/day):                                                │
│  • Viewing events (what everyone watches)                                │
│  • A/B test events (experiment results)                                  │
│  • Infrastructure logs (server health)                                   │
│  • Content metadata (movie info)                                         │
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────────┐                                                     │
│  │   Apache Kafka    │  (Event streaming - real-time)                     │
│  └────────┬────────┘                                                     │
│           │                                                              │
│     ┌─────┴──────────────────────────────────┐                           │
│     ▼                                        ▼                           │
│  ┌──────────────────┐              ┌──────────────────┐                  │
│  │  Data Lake (S3)   │              │  Real-Time Layer  │                  │
│  │  ─────────────    │              │  (Druid/Flink)    │                  │
│  │  • Raw events     │              │  Live dashboards  │                  │
│  │  • Parquet/Iceberg│              └──────────────────┘                  │
│  │  • Petabytes      │                                                   │
│  └────────┬─────────┘                                                    │
│           │                                                              │
│           │  Apache Spark (Transform)                                    │
│           ▼                                                              │
│  ┌──────────────────┐                                                    │
│  │  Redshift/Presto  │  (SQL queries for analysts)                       │
│  │  (Warehouse)      │                                                   │
│  └────────┬─────────┘                                                    │
│           │                                                              │
│           ▼                                                              │
│  ┌──────────────────┐                                                    │
│  │  Recommendation   │  (ML models trained on lake data)                 │
│  │  Engine           │  "Because you watched..."                         │
│  └──────────────────┘                                                    │
│                                                                          │
│  Key: Netflix stores ALL raw data in the lake (cheap S3)                 │
│       Curated/aggregated data goes to warehouse (fast queries)           │
│       ML training reads directly from the lake                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Uber

- **Data Lake**: 100+ PB on HDFS → migrating to Apache Hudi on object storage
- **Data Warehouse**: Presto/Trino queries against the lake + Vertica for BI
- **Scale**: 10,000+ Spark jobs daily, 100+ PB total data
- **Innovation**: Built Apache Hudi (open source) to support updates on data lakes

### LinkedIn

- **Data Lake**: Exabytes stored on HDFS and Azure ADLS
- **Warehouse**: Trino for interactive queries, Spark for batch
- **Unique**: Built their own "Pinot" real-time OLAP system for serving analytics at LinkedIn scale

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Data Lake becomes "Data Swamp" | No metadata, no catalog, nobody knows what's there | Use a Data Catalog (AWS Glue, Unity Catalog), enforce naming conventions |
| Over-storing in the warehouse | Paying warehouse prices for data nobody queries | Store raw in lake (cheap), only load what's needed into warehouse |
| No partitioning in the lake | Full scans of entire dataset every query | Partition by date, region, or other high-cardinality dimension |
| Wrong file format in lake | CSV is slow, JSON wastes space | Use Parquet or ORC for analytical workloads |
| Ignoring file sizes | Too many small files → metadata overhead | Compact files to 128MB-1GB range |
| No access controls on lake | Everyone can read everything (PII exposure) | Implement column/row-level security, encryption |
| Choosing warehouse before understanding needs | Locked into wrong vendor | Evaluate: query patterns, data volume, team skills, cloud provider |

---

## When to Use / When NOT to Use

### Use a Data Lake When:
- ✅ You need to store raw, unstructured data (logs, images, sensor data)
- ✅ Storage cost is the primary concern (pennies per GB)
- ✅ Data scientists need raw data for ML model training
- ✅ You don't yet know all the questions you'll ask of the data
- ✅ Data volume is massive (petabytes+)

### Use a Data Warehouse When:
- ✅ Business analysts need fast, interactive SQL queries
- ✅ BI tools (Looker, Tableau) need a structured backend
- ✅ You need governance, access controls, and audit trails
- ✅ Query performance matters more than storage cost
- ✅ Data is structured and well-understood

### Use a Lakehouse When:
- ✅ You need both ML/data science (raw data) AND BI (structured queries)
- ✅ You want to avoid data duplication between lake and warehouse
- ✅ You need ACID transactions on your data lake (updates, deletes)
- ✅ You're on Databricks, or using Delta Lake/Iceberg/Hudi

### Consider NOT Using Either When:
- ❌ You have less than 100GB of data — PostgreSQL might be enough
- ❌ You don't have analysts or data scientists who will query it
- ❌ You're building a data lake "just in case" with no users in mind

---

## Key Takeaways

1. **Data Lakes** store everything in raw format on cheap object storage (S3). Great for ML and archival, but slow for SQL queries without additional tooling.

2. **Data Warehouses** (BigQuery, Snowflake, Redshift) store structured data in optimized columnar format. Fast SQL queries, but more expensive and requires schema upfront.

3. **Separation of storage and compute** is the key architectural innovation — it lets you scale reads independently of storage size.

4. **The Lakehouse** (Delta Lake, Iceberg, Hudi) brings warehouse-like features (ACID, schema, fast queries) to data lakes. It's the direction the industry is heading.

5. **Partitioning** (by date, region, etc.) is critical for performance in both lakes and warehouses — it allows query engines to skip irrelevant data.

6. Most companies at scale use **both**: raw data in the lake, curated data in the warehouse, with pipelines connecting them.

7. **Choose based on your users**: ML engineers → Lake. Business analysts → Warehouse. Both → Lakehouse.

---

## What's Next?

We've covered where analytical data lives. But there's a critical piece of the puzzle: **how do you continuously capture changes from your OLTP databases and stream them to your lake/warehouse in real-time?** That's [Chapter 20.4: Change Data Capture (CDC)](./04-change-data-capture.md) — the technology that bridges the gap between your live application and your analytical systems.
