# Chapter 53: Amazon Athena & AWS Glue

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Amazon Athena Fundamentals](#part-1-amazon-athena-fundamentals)
- [Part 2: Running Queries in Athena (Full Portal Walkthrough)](#part-2-running-queries-in-athena-full-portal-walkthrough)
- [Part 3: AWS Glue Data Catalog & Crawlers](#part-3-aws-glue-data-catalog--crawlers)
- [Part 4: AWS Glue ETL Jobs](#part-4-aws-glue-etl-jobs)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What Are Athena and Glue? Why Do We Need a Data Lake?

Imagine you have terabytes of CSV, JSON, and Parquet files stored in S3 — website logs, customer data, sales records. You want to run SQL queries on this data without setting up a database or loading the data anywhere.

**Athena is like running SQL directly on your S3 files.** No database to set up, no data to load — just point Athena at your S3 bucket, tell it the file format, and start querying with standard SQL. You pay only for the data scanned by each query.

**Glue** is the companion service that:
- **Crawls** your S3 data to automatically discover its schema ("this CSV has columns: name, email, signup_date")
- **Transforms (ETL)** raw data into clean, optimized formats (e.g., convert messy CSV → efficient Parquet)
- **Catalogs** everything in the **Data Catalog** so Athena, Redshift, and EMR can find and query it

**What is a Data Lake?** Think of it as a giant storage pool (S3) where you dump ALL your raw data — structured, semi-structured, unstructured — and then use tools like Athena and Glue to make sense of it later. Unlike a database, you don't need to define the schema upfront.

Amazon Athena is a serverless SQL query service that analyzes data directly in S3. AWS Glue provides a serverless ETL service with a centralized Data Catalog. Together they form the foundation of a serverless data lake.

```
What you'll learn:
├── Athena fundamentals (serverless SQL on S3)
├── Running queries in Athena (portal walkthrough)
├── Glue Data Catalog & Crawlers
├── Glue ETL jobs (PySpark/Python)
├── Performance optimization (partitioning, formats)
└── Terraform & CLI examples
```

---

## Part 1: Amazon Athena Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW ATHENA WORKS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ S3 (data) + Glue Data Catalog (schema) → Athena (SQL queries)     │
│                                                                       │
│ ┌──────────┐     ┌──────────┐     ┌──────────┐                    │
│ │ S3 Data  │     │  Glue    │     │  Athena  │                    │
│ │ (CSV,    │────▶│  Data    │────▶│  (SQL    │                    │
│ │  JSON,   │     │ Catalog  │     │  engine) │                    │
│ │  Parquet,│     │ (schema) │     │          │                    │
│ │  ORC)    │     └──────────┘     └──────────┘                    │
│ └──────────┘                                                        │
│                                                                       │
│ Key features:                                                        │
│ ├── Serverless: No infrastructure to manage                     │
│ ├── Standard SQL: Presto/Trino engine                           │
│ ├── Pay per query: $5/TB of data scanned                        │
│ ├── Federated queries: Query across S3, DynamoDB, RDS, etc.   │
│ ├── Views, CTAS (Create Table As Select)                       │
│ ├── Prepared statements (parameterized queries)                 │
│ └── Workgroups for cost/access control                          │
│                                                                       │
│ Supported data formats:                                              │
│ ├── CSV, TSV (text)                                              │
│ ├── JSON, JSON Lines                                             │
│ ├── Apache Parquet (⚡ columnar, best for analytics)            │
│ ├── Apache ORC (columnar)                                       │
│ ├── Avro                                                         │
│ └── CloudTrail logs, VPC Flow Logs, ALB logs                   │
│                                                                       │
│ Performance & cost optimization:                                    │
│ ├── Use Parquet/ORC: Columnar = scan only needed columns       │
│ ├── Partition data: year/month/day → scan only relevant data  │
│ ├── Compress: GZIP, Snappy, LZ4, ZSTD                         │
│ ├── Use CTAS to convert CSV to Parquet                          │
│ └── ⚡ Partitioned Parquet with compression = cheapest queries │
│                                                                       │
│ Pricing:                                                             │
│ ├── $5.00 per TB of data scanned                                │
│ ├── $0 if no data scanned (DDL, failed queries)                │
│ ├── Minimum charge: 10 MB per query                             │
│ └── ⚡ Parquet reduces cost by 30-90% vs CSV                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Running Queries in Athena (Full Portal Walkthrough)

```
Console → Athena → Query editor

┌─────────────────────────────────────────────────────────────────┐
│           ATHENA SETUP                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ First-time setup:                                              │
│ Settings → Manage → Query result location:                    │
│ [s3://athena-results-123456/query-results/]                  │
│ → ⚡ Required: S3 location for query results                 │
│ → Encrypt query results: ☑ SSE-S3                           │
│                                                                   │
│ Workgroup: [primary ▼]                                        │
│ → Controls: Query result location, data scan limits, tags   │
│ → ⚡ Create workgroups per team for cost tracking            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           CREATING A TABLE (pointing to S3 data)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Database: [default ▼] or create new                           │
│                                                                   │
│ -- Create database                                             │
│ CREATE DATABASE IF NOT EXISTS analytics;                       │
│                                                                   │
│ -- Create table (CSV data in S3)                              │
│ CREATE EXTERNAL TABLE analytics.clickstream (                 │
│   event_id STRING,                                            │
│   user_id STRING,                                             │
│   page STRING,                                                │
│   action STRING,                                              │
│   timestamp TIMESTAMP                                         │
│ )                                                              │
│ PARTITIONED BY (year STRING, month STRING, day STRING)        │
│ ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'│
│ LOCATION 's3://clickstream-data/events/'                     │
│ TBLPROPERTIES ('has_encrypted_data'='false');                 │
│                                                                   │
│ -- Load partitions                                             │
│ MSCK REPAIR TABLE analytics.clickstream;                      │
│ → Discovers partitions from S3 folder structure              │
│ → Or use: ALTER TABLE ADD PARTITION ...                      │
│                                                                   │
│ -- Parquet table (⚡ much faster & cheaper)                   │
│ CREATE EXTERNAL TABLE analytics.clickstream_parquet (         │
│   event_id STRING,                                            │
│   user_id STRING,                                             │
│   page STRING,                                                │
│   action STRING,                                              │
│   timestamp TIMESTAMP                                         │
│ )                                                              │
│ PARTITIONED BY (year STRING, month STRING, day STRING)        │
│ STORED AS PARQUET                                              │
│ LOCATION 's3://clickstream-data/events-parquet/'             │
│ TBLPROPERTIES ('parquet.compression'='SNAPPY');              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           RUNNING QUERIES                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Query editor:                                                  │
│ ┌────────────────────────────────────────────────────┐        │
│ │ SELECT page, COUNT(*) as clicks                   │        │
│ │ FROM analytics.clickstream                        │        │
│ │ WHERE year = '2024' AND month = '01'              │        │
│ │ GROUP BY page                                      │        │
│ │ ORDER BY clicks DESC                               │        │
│ │ LIMIT 10;                                          │        │
│ └────────────────────────────────────────────────────┘        │
│                                                                   │
│ [Run] → Results appear below                                 │
│                                                                   │
│ Query stats:                                                   │
│ ├── Run time: 2.3 seconds                                   │
│ ├── Data scanned: 145 MB (with partitioning)               │
│ │   vs 12 GB without partitioning                          │
│ └── Cost: $0.000725 (vs $0.06 without partitioning)       │
│                                                                   │
│ -- Convert CSV to Parquet (CTAS)                              │
│ CREATE TABLE analytics.clickstream_parquet                    │
│ WITH (                                                        │
│   format = 'PARQUET',                                        │
│   parquet_compression = 'SNAPPY',                            │
│   partitioned_by = ARRAY['year', 'month', 'day'],           │
│   external_location = 's3://bucket/events-parquet/'         │
│ ) AS                                                          │
│ SELECT * FROM analytics.clickstream;                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: AWS Glue Data Catalog & Crawlers

```
┌─────────────────────────────────────────────────────────────────────┐
│           GLUE DATA CATALOG                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Centralized metadata repository (schema registry)            │
│ → Used by: Athena, Redshift Spectrum, EMR, Glue ETL              │
│                                                                       │
│ Components:                                                          │
│ ├── Databases: Logical grouping of tables                       │
│ ├── Tables: Schema definition pointing to S3 data              │
│ ├── Partitions: Data subdivision (year/month/day)              │
│ └── Connections: JDBC connections to RDS, Redshift, etc.       │
│                                                                       │
│ ⚡ Glue Data Catalog = Apache Hive Metastore compatible          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Console → AWS Glue → Crawlers → Add crawler

┌─────────────────────────────────────────────────────────────────┐
│           CREATE CRAWLER                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [clickstream-crawler]                                   │
│                                                                   │
│ Data sources:                                                  │
│ [Add a data source]                                           │
│ ├── Data source: [S3 ▼]                                     │
│ ├── S3 path: [s3://clickstream-data/events/]                │
│ └── Subsequent crawl behavior:                               │
│     ● Crawl all sub-folders                                  │
│     ○ Crawl new sub-folders only                             │
│                                                                   │
│ IAM role:                                                      │
│ ● Create new IAM role                                         │
│ ○ Choose existing: [AWSGlueServiceRole-Crawler ▼]           │
│                                                                   │
│ Target database: [analytics ▼]                                │
│ Table name prefix: [raw_]                                     │
│                                                                   │
│ Crawler schedule:                                              │
│ ● Run on demand                                               │
│ ○ Daily: [cron(0 8 * * ? *)]                                │
│ ○ Hourly                                                      │
│ ○ Custom cron                                                │
│ → ⚡ Schedule if data changes frequently                    │
│                                                                   │
│ Schema changes:                                                │
│ ● Update the table definition in the data catalog            │
│ ○ Add new columns only                                       │
│ ○ Log changes and don't update                              │
│                                                                   │
│ Output:                                                        │
│ → Creates/updates Glue Data Catalog tables                  │
│ → Auto-discovers: schema, partitions, data format           │
│ → Tables immediately available in Athena                    │
│                                                                   │
│                    [Create crawler → Run crawler]             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: AWS Glue ETL Jobs

```
Console → AWS Glue → ETL jobs → Visual ETL

┌─────────────────────────────────────────────────────────────────┐
│           CREATE GLUE ETL JOB                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Job type:                                                      │
│ ● Visual ETL (drag-and-drop, ⚡ easiest)                    │
│ ○ Spark script editor (PySpark / Scala)                     │
│ ○ Python Shell script editor                                 │
│ ○ Jupyter Notebook                                           │
│                                                                   │
│ Visual ETL canvas:                                            │
│                                                                   │
│ [Source] → [Transform] → [Target]                            │
│                                                                   │
│ Sources:                                                       │
│ ├── S3 (Glue Data Catalog table)                            │
│ ├── JDBC (RDS, Redshift, etc.)                              │
│ ├── Kinesis Data Stream                                     │
│ └── Kafka                                                    │
│                                                                   │
│ Transforms:                                                    │
│ ├── ApplyMapping: Rename/retype columns                     │
│ ├── Filter: Filter rows by condition                        │
│ ├── Join: Merge two datasets                                │
│ ├── SelectFields: Pick specific columns                     │
│ ├── DropFields: Remove columns                              │
│ ├── DropDuplicates: Remove duplicate rows                   │
│ ├── FillMissing: Handle null values                         │
│ ├── Aggregate: GROUP BY operations                          │
│ ├── Custom transform: Write PySpark code                    │
│ └── SQL: Write SQL transformations                          │
│                                                                   │
│ Targets:                                                       │
│ ├── S3 (Parquet, CSV, JSON, etc.)                          │
│ ├── Glue Data Catalog (update catalog tables)              │
│ └── JDBC (write to RDS, Redshift)                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           JOB SETTINGS                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Job name: [csv-to-parquet-daily]                              │
│                                                                   │
│ IAM Role: [AWSGlueServiceRole-ETL ▼]                        │
│                                                                   │
│ Glue version: [Glue 4.0 ▼] (Spark 3.3, Python 3.10)       │
│                                                                   │
│ Language: ● Python ○ Scala                                   │
│                                                                   │
│ Worker type:                                                   │
│ ● G.1X (4 vCPU, 16 GB) → Standard                          │
│ ○ G.2X (8 vCPU, 32 GB) → Memory-intensive                 │
│ ○ G.4X (16 vCPU, 64 GB) → Large datasets                  │
│ ○ G.8X (32 vCPU, 128 GB) → Very large                     │
│                                                                   │
│ Number of workers: [10] (min 2)                              │
│ → ⚡ Start small, increase if job is slow                   │
│                                                                   │
│ Job timeout: [2880] minutes (48 hours max)                   │
│                                                                   │
│ Job bookmark: ☑ Enabled                                      │
│ → Tracks what data has been processed                       │
│ → ⚡ Essential for incremental processing                   │
│ → Only processes new data on subsequent runs               │
│                                                                   │
│ Retry attempts: [1]                                           │
│                                                                   │
│ Schedule:                                                      │
│ [cron(0 2 * * ? *)] (2 AM daily)                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# Glue database
resource "aws_glue_catalog_database" "analytics" {
  name = "analytics"
}

# Glue crawler
resource "aws_glue_crawler" "clickstream" {
  name          = "clickstream-crawler"
  database_name = aws_glue_catalog_database.analytics.name
  role          = aws_iam_role.glue.arn
  schedule      = "cron(0 8 * * ? *)"

  s3_target {
    path = "s3://clickstream-data/events/"
  }

  schema_change_policy {
    update_behavior = "UPDATE_IN_DATABASE"
    delete_behavior = "LOG"
  }

  configuration = jsonencode({
    Version = 1.0
    Grouping = { TableGroupingPolicy = "CombineCompatibleSchemas" }
  })
}

# Glue ETL job
resource "aws_glue_job" "etl" {
  name     = "csv-to-parquet"
  role_arn = aws_iam_role.glue.arn

  command {
    name            = "glueetl"
    script_location = "s3://glue-scripts/csv-to-parquet.py"
    python_version  = "3"
  }

  glue_version      = "4.0"
  number_of_workers = 10
  worker_type       = "G.1X"
  timeout           = 120
  max_retries       = 1

  default_arguments = {
    "--job-bookmark-option"         = "job-bookmark-enable"
    "--TempDir"                     = "s3://glue-temp/temp/"
    "--enable-continuous-cloudwatch-log" = "true"
    "--source_database"             = "analytics"
    "--source_table"                = "raw_clickstream"
    "--target_path"                 = "s3://clickstream-data/events-parquet/"
  }
}

# Athena workgroup
resource "aws_athena_workgroup" "analytics" {
  name = "analytics-team"

  configuration {
    result_configuration {
      output_location = "s3://athena-results-123456/analytics/"
      encryption_configuration {
        encryption_option = "SSE_S3"
      }
    }
    bytes_scanned_cutoff_per_query = 10737418240  # 10 GB limit
    enforce_workgroup_configuration = true
  }
}

# Athena named query
resource "aws_athena_named_query" "top_pages" {
  name      = "top-pages-daily"
  database  = aws_glue_catalog_database.analytics.name
  workgroup = aws_athena_workgroup.analytics.name
  query     = <<-SQL
    SELECT page, COUNT(*) as views
    FROM clickstream
    WHERE year = CAST(YEAR(CURRENT_DATE) AS VARCHAR)
      AND month = LPAD(CAST(MONTH(CURRENT_DATE) AS VARCHAR), 2, '0')
    GROUP BY page
    ORDER BY views DESC
    LIMIT 20
  SQL
}
```

```bash
# Run Athena query
aws athena start-query-execution \
  --query-string "SELECT * FROM analytics.clickstream LIMIT 10" \
  --work-group analytics-team \
  --result-configuration OutputLocation=s3://athena-results/

# Get query results
aws athena get-query-results --query-execution-id query-id

# Run Glue crawler
aws glue start-crawler --name clickstream-crawler

# Run Glue job
aws glue start-job-run --job-name csv-to-parquet

# Get Glue tables
aws glue get-tables --database-name analytics
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Serverless Data Lake                                     │
│ Raw Data (S3) → Glue Crawler → Glue ETL → Parquet (S3)          │
│                                           → Athena (SQL queries)  │
│ ├── Glue Crawler auto-discovers schema                           │
│ ├── Glue ETL converts CSV → Parquet (daily)                    │
│ ├── Athena queries optimized Parquet data                       │
│ └── ⚡ No servers, pay-per-query                                │
│                                                                       │
│ Pattern 2: Log Analytics                                             │
│ CloudTrail / VPC Flow / ALB Logs → S3 → Athena                   │
│ ├── AWS provides pre-built table definitions                    │
│ ├── Partitioned by date automatically                            │
│ ├── Query security events, traffic patterns, errors            │
│ └── ⚡ No ETL needed for AWS native logs                       │
│                                                                       │
│ Pattern 3: Data Pipeline (ETL)                                      │
│ RDS (JDBC) → Glue ETL → S3 (Parquet) → Athena + QuickSight    │
│ ├── Extract from operational database                            │
│ ├── Transform: Clean, aggregate, denormalize                   │
│ ├── Load: Write partitioned Parquet to S3                      │
│ ├── Job bookmark: Process only new records                     │
│ └── Schedule: Nightly ETL run                                    │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Always use Parquet/ORC (columnar) for analytics data        │
│ 2. Partition by date (year/month/day) for cost savings        │
│ 3. Enable Glue job bookmarks for incremental processing       │
│ 4. Use Athena workgroups for cost control per team             │
│ 5. Set byte scan limits to prevent runaway queries             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common Athena & Glue Issues

### "HIVE_CURSOR_ERROR" or "HIVE_BAD_DATA"

```
Cause: Schema mismatch between Glue Data Catalog and actual S3 data

Fixes:
1. Re-run Glue Crawler to update schema
2. Check data format matches table definition (CSV vs JSON vs Parquet)
3. Check for corrupt files in S3 path
4. Verify SerDe (serialization format) is correct
```

### "My Athena query is scanning too much data (expensive!)"

```
Cost: $5 per TB scanned

Optimization:
1. Use columnar formats (Parquet/ORC instead of CSV/JSON)
   → Parquet only reads columns you SELECT (not all columns)
   → Can reduce scan by 90%+

2. Partition your data:
   S3: s3://data/year=2025/month=01/day=15/
   Query: WHERE year = '2025' AND month = '01'
   → Only scans matching partitions

3. Use LIMIT for exploration (but still scans all matching data!)
   To truly limit scans, use partitions + columnar formats

4. Compress data (gzip, snappy, zstd)
   → Less data to scan = lower cost
```

### "Glue Crawler creates wrong schema"

```
1. Crawler merges schemas across files — inconsistent files cause issues
2. Fix: Ensure all files in a path have the same schema
3. Use classifiers to tell the crawler exactly how to parse
4. Manual alternative: Skip the crawler, define table schema manually
   Console → Glue → Data Catalog → Tables → Add table manually
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Querying CSV/JSON (not Parquet) | 10-100x more expensive | Convert with Glue ETL or CTAS |
| No partitions | Full table scan every query | Partition by date/region/category |
| Crawler runs too often | Unnecessary Glue charges | Schedule crawler weekly or on-demand |
| No query result location set | Queries fail | Set S3 output location in Athena settings |
| Not using workgroups | Can't control per-team costs | Create workgroups with data scan limits |

---

## Quick Reference

```
Athena & Glue Quick Reference:
├── Athena: Serverless SQL on S3 ($5/TB scanned)
│   ├── Engine: Presto/Trino (standard SQL)
│   ├── Best format: Parquet/ORC (columnar, compressed)
│   ├── Partitioning: Reduces data scanned (= cost savings)
│   ├── CTAS: Convert CSV → Parquet in-place
│   └── Workgroups: Cost control + access management
├── Glue Data Catalog: Centralized metadata (schema registry)
│   ├── Databases + Tables + Partitions
│   ├── Used by: Athena, Redshift Spectrum, EMR
│   └── Hive Metastore compatible
├── Glue Crawlers: Auto-discover schema from S3/JDBC data
├── Glue ETL: Serverless Spark jobs (Python/Scala)
│   ├── Visual ETL (drag-and-drop) or code
│   ├── Job bookmarks for incremental processing
│   └── Workers: G.1X to G.8X
├── ⚡ Partitioned Parquet + compression = cheapest queries
├── ⚡ Glue Crawler → Glue ETL → Athena = serverless data lake
└── ⚡ Use Athena for ad-hoc queries, Glue for scheduled ETL
```

---

## What's Next?

In **Chapter 54: Amazon Redshift**, we'll cover the cloud data warehouse, cluster creation, Redshift Serverless, Spectrum for querying S3, and data loading patterns.
