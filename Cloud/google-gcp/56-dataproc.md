# Chapter 56 — Dataproc

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Dataproc Fundamentals](#part-1--dataproc-fundamentals)
- [Part 2: Cluster Architecture](#part-2--cluster-architecture)
- [Part 3: Cluster Creation & Configuration](#part-3--cluster-creation--configuration)
- [Part 4: Submitting Jobs](#part-4--submitting-jobs)
- [Part 5: Spark on Dataproc](#part-5--spark-on-dataproc)
- [Part 6: Hadoop & Hive on Dataproc](#part-6--hadoop--hive-on-dataproc)
- [Part 7: Autoscaling](#part-7--autoscaling)
- [Part 8: Dataproc Serverless](#part-8--dataproc-serverless)
- [Part 9: Initialization Actions & Components](#part-9--initialization-actions--components)
- [Part 10: Metastore Service](#part-10--metastore-service)
- [Part 11: Persistent History Server](#part-11--persistent-history-server)
- [Part 12: Security & Networking](#part-12--security--networking)
- [Part 13: Monitoring & Troubleshooting](#part-13--monitoring--troubleshooting)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Dataproc is Google Cloud's fully managed service for running Apache Spark, Hadoop, Hive, Pig, Presto, and other open-source data analytics frameworks. Clusters can be created in 90 seconds, auto-scale based on workload, and be deleted after jobs complete — paying only for what you use. Dataproc Serverless eliminates cluster management entirely, letting you submit Spark jobs directly.

---

## Part 1 — Dataproc Fundamentals

### What Is Dataproc?

```
┌────────────────────────────────────────────────────────────────────┐
│         DATAPROC OVERVIEW                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Managed Hadoop/Spark ecosystem on GCP:                            │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Dataproc Cluster                                         │     │
│  │                                                            │     │
│  │  Pre-installed frameworks:                                │     │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │     │
│  │  │ Spark  │ │ Hadoop │ │ Hive   │ │ Pig    │           │     │
│  │  │ (3.5)  │ │ (3.3)  │ │ (3.1)  │ │        │           │     │
│  │  └────────┘ └────────┘ └────────┘ └────────┘           │     │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │     │
│  │  │ Presto │ │ Flink  │ │ Jupyter│ │ Zeppelin│          │     │
│  │  │        │ │        │ │        │ │        │           │     │
│  │  └────────┘ └────────┘ └────────┘ └────────┘           │     │
│  │                                                            │     │
│  │  Key advantages over self-managed:                        │     │
│  │  • 90-second cluster creation                            │     │
│  │  • Auto-scaling (add/remove workers)                     │     │
│  │  • Pay per second of usage                               │     │
│  │  • Ephemeral clusters (create → run → delete)           │     │
│  │  • GCS as primary storage (not HDFS)                     │     │
│  │  • Native BigQuery connector                             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Deployment modes:                                                  │
│  • Dataproc on Compute Engine (clusters)                          │
│  • Dataproc on GKE (Kubernetes)                                   │
│  • Dataproc Serverless (no cluster management)                    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Dataproc | AWS EMR | Azure HDInsight |
|---------|-------------|---------|-----------------|
| Managed Spark/Hadoop | Dataproc | EMR | HDInsight |
| Serverless Spark | Dataproc Serverless | EMR Serverless | Synapse Spark |
| Cluster creation | ~90 seconds | ~5-10 minutes | ~15-20 minutes |
| Pricing | Per-second + VM cost | Per-second + VM cost | Per-hour + VM cost |
| Dataproc premium | $0.01/vCPU-hr | $0.015/vCPU-hr (EMR) | Included |
| GCS/S3 integration | Native (GCS connector) | Native (EMRFS) | WASB/ABFS |
| Metastore | Dataproc Metastore | Glue Data Catalog | External HMS |
| Spot/Preemptible | Yes | Yes (Spot) | Yes |

### Pricing

| Component | Cost |
|-----------|------|
| Dataproc premium | $0.01/vCPU-hr (on top of VM cost) |
| Compute VMs | Standard Compute Engine pricing |
| Spot VMs | 60-91% discount on VM cost |
| Dataproc Serverless | $0.06/DCU-hr (Data Compute Unit) |
| Persistent disk | Standard disk pricing |
| GCS storage | Standard GCS pricing |

---

## Part 2 — Cluster Architecture

### Cluster Topology

```
┌────────────────────────────────────────────────────────────────────┐
│         CLUSTER ARCHITECTURE                                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  MASTER NODE(S)                                            │     │
│  │  ┌──────────────┐  ┌──────────────┐                      │     │
│  │  │ Master 1     │  │ Master 2     │  (HA: 3 masters)    │     │
│  │  │ NameNode     │  │ NameNode(SB) │                      │     │
│  │  │ YARN RM      │  │ YARN RM(SB)  │                      │     │
│  │  │ Spark Master │  │              │                      │     │
│  │  │ Hive Server  │  │              │                      │     │
│  │  └──────────────┘  └──────────────┘                      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  PRIMARY WORKER NODES (required, persistent)              │     │
│  │  ┌────────┐ ┌────────┐ ┌────────┐                       │     │
│  │  │Worker 1│ │Worker 2│ │Worker 3│   On-demand VMs       │     │
│  │  │DataNode│ │DataNode│ │DataNode│   Run YARN tasks      │     │
│  │  │NodeMgr │ │NodeMgr │ │NodeMgr │   Have local HDFS     │     │
│  │  └────────┘ └────────┘ └────────┘                       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  SECONDARY WORKER NODES (optional, auto-scale)            │     │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │     │
│  │  │Worker 4│ │Worker 5│ │Worker 6│ │Worker 7│           │     │
│  │  │Spot VM │ │Spot VM │ │Spot VM │ │Spot VM │           │     │
│  │  │No HDFS │ │No HDFS │ │No HDFS │ │No HDFS │           │     │
│  │  └────────┘ └────────┘ └────────┘ └────────┘           │     │
│  │  Preemptible — can be reclaimed anytime                  │     │
│  │  No HDFS data — only compute                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Best practice: Use GCS instead of HDFS                            │
│  → Decouple storage from compute                                  │
│  → Data persists after cluster deletion                            │
│  → Multiple clusters can share same data                          │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 3 — Cluster Creation & Configuration

### Create a Cluster

```bash
# Basic cluster
gcloud dataproc clusters create my-cluster \
    --region=us-central1 \
    --zone=us-central1-a \
    --master-machine-type=n2-standard-4 \
    --master-boot-disk-size=100GB \
    --num-workers=3 \
    --worker-machine-type=n2-standard-4 \
    --worker-boot-disk-size=200GB \
    --image-version=2.1-debian11

# HA cluster (3 masters)
gcloud dataproc clusters create ha-cluster \
    --region=us-central1 \
    --num-masters=3 \
    --master-machine-type=n2-standard-4 \
    --num-workers=5 \
    --worker-machine-type=n2-standard-8

# With secondary (preemptible/spot) workers
gcloud dataproc clusters create cost-cluster \
    --region=us-central1 \
    --num-workers=2 \
    --worker-machine-type=n2-standard-4 \
    --num-secondary-workers=10 \
    --secondary-worker-type=spot

# With optional components
gcloud dataproc clusters create ml-cluster \
    --region=us-central1 \
    --optional-components=JUPYTER,DOCKER \
    --enable-component-gateway \
    --num-workers=4 \
    --worker-machine-type=n2-standard-8

# With autoscaling policy
gcloud dataproc clusters create auto-cluster \
    --region=us-central1 \
    --autoscaling-policy=my-policy \
    --num-workers=2 \
    --worker-machine-type=n2-standard-4
```

### Image Versions

| Version | Spark | Hadoop | Hive | Python |
|---------|-------|--------|------|--------|
| 2.2 | 3.5 | 3.3 | 3.1 | 3.11 |
| 2.1 | 3.3 | 3.3 | 3.1 | 3.10 |
| 2.0 | 3.1 | 3.2 | 3.1 | 3.8 |

---

## Part 4 — Submitting Jobs

### Job Types

```bash
# PySpark job
gcloud dataproc jobs submit pyspark \
    gs://my-bucket/scripts/etl.py \
    --cluster=my-cluster \
    --region=us-central1 \
    --jars=gs://spark-lib/bigquery/spark-bigquery-latest_2.12.jar \
    -- --input=gs://data-bucket/raw/ --output=gs://data-bucket/processed/

# Spark (Scala/Java) job
gcloud dataproc jobs submit spark \
    --cluster=my-cluster \
    --region=us-central1 \
    --class=com.mycompany.ETLJob \
    --jars=gs://my-bucket/jars/etl-1.0.jar \
    -- --date=2024-01-15

# Hive job
gcloud dataproc jobs submit hive \
    --cluster=my-cluster \
    --region=us-central1 \
    --file=gs://my-bucket/queries/report.hql

# SparkSQL job
gcloud dataproc jobs submit spark-sql \
    --cluster=my-cluster \
    --region=us-central1 \
    --file=gs://my-bucket/queries/transform.sql

# Spark-R job
gcloud dataproc jobs submit spark-r \
    gs://my-bucket/scripts/analysis.R \
    --cluster=my-cluster \
    --region=us-central1

# Hadoop MapReduce job
gcloud dataproc jobs submit hadoop \
    --cluster=my-cluster \
    --region=us-central1 \
    --class=org.apache.hadoop.examples.WordCount \
    --jars=file:///usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar \
    -- gs://input-bucket/ gs://output-bucket/

# Pig job
gcloud dataproc jobs submit pig \
    --cluster=my-cluster \
    --region=us-central1 \
    --file=gs://my-bucket/scripts/analysis.pig
```

---

## Part 5 — Spark on Dataproc

### PySpark Example

```python
# etl.py — PySpark ETL job
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder \
    .appName("Sales ETL") \
    .getOrCreate()

# Read from GCS (Parquet)
df = spark.read.parquet("gs://data-bucket/raw/sales/")

# Transform
result = (
    df
    .filter(F.col("amount") > 0)
    .withColumn("year", F.year("order_date"))
    .withColumn("month", F.month("order_date"))
    .groupBy("year", "month", "region")
    .agg(
        F.sum("amount").alias("total_revenue"),
        F.count("order_id").alias("order_count"),
        F.avg("amount").alias("avg_order_value"),
    )
)

# Write to BigQuery
result.write \
    .format("bigquery") \
    .option("table", "my-project.analytics.monthly_revenue") \
    .option("temporaryGcsBucket", "my-temp-bucket") \
    .mode("overwrite") \
    .save()

# Write to GCS (Parquet, partitioned)
result.write \
    .partitionBy("year", "month") \
    .parquet("gs://data-bucket/processed/monthly_revenue/")

spark.stop()
```

### Spark BigQuery Connector

```bash
# Submit with BigQuery connector
gcloud dataproc jobs submit pyspark etl.py \
    --cluster=my-cluster \
    --region=us-central1 \
    --jars=gs://spark-lib/bigquery/spark-bigquery-with-dependencies_2.12-0.36.1.jar \
    --properties=spark.executor.memory=4g,spark.executor.cores=2
```

```python
# Read from BigQuery in Spark
df = spark.read \
    .format("bigquery") \
    .option("table", "my-project.analytics.events") \
    .option("filter", "event_date = '2024-01-15'") \
    .load()
```

---

## Part 6 — Hadoop & Hive on Dataproc

### Hive Queries

```sql
-- report.hql — Hive query file
-- Use GCS as external storage (not HDFS)

CREATE EXTERNAL TABLE IF NOT EXISTS sales (
    order_id STRING,
    customer_id STRING,
    product STRING,
    amount DOUBLE,
    order_date DATE
)
STORED AS PARQUET
LOCATION 'gs://data-bucket/raw/sales/';

-- Create partitioned summary table
CREATE TABLE IF NOT EXISTS monthly_sales (
    region STRING,
    total_revenue DOUBLE,
    order_count INT
)
PARTITIONED BY (year INT, month INT)
STORED AS PARQUET
LOCATION 'gs://data-bucket/processed/monthly_sales/';

-- Insert with dynamic partitioning
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT OVERWRITE TABLE monthly_sales PARTITION(year, month)
SELECT
    region,
    SUM(amount) AS total_revenue,
    COUNT(*) AS order_count,
    YEAR(order_date) AS year,
    MONTH(order_date) AS month
FROM sales
GROUP BY region, YEAR(order_date), MONTH(order_date);
```

---

## Part 7 — Autoscaling

### Autoscaling Policies

```bash
# Create autoscaling policy
gcloud dataproc autoscaling-policies create my-policy \
    --region=us-central1 \
    --max-workers=20 \
    --min-workers=2 \
    --scale-up-factor=1.0 \
    --scale-down-factor=1.0 \
    --cooldown-period=2m \
    --scale-up-min-worker-fraction=0.0 \
    --scale-down-min-worker-fraction=0.0

# Create cluster with autoscaling
gcloud dataproc clusters create my-cluster \
    --region=us-central1 \
    --autoscaling-policy=my-policy \
    --num-workers=2
```

```yaml
# autoscaling-policy.yaml
workerConfig:
  minInstances: 2
  maxInstances: 10
  weight: 1
secondaryWorkerConfig:
  minInstances: 0
  maxInstances: 50
  weight: 1
basicAlgorithm:
  yarnConfig:
    scaleUpFactor: 1.0
    scaleDownFactor: 1.0
    scaleUpMinWorkerFraction: 0.0
    scaleDownMinWorkerFraction: 0.0
    cooldownPeriod: 2m
    gracefulDecommissionTimeout: 1h
```

---

## Part 8 — Dataproc Serverless

### Submit Without a Cluster

```
┌────────────────────────────────────────────────────────────────────┐
│         DATAPROC SERVERLESS                                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  No cluster to create or manage:                                   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  You submit: PySpark script + parameters                  │     │
│  │  Dataproc:   provisions resources, runs job, cleans up   │     │
│  │                                                            │     │
│  │  Benefits:                                                │     │
│  │  • Zero cluster management                               │     │
│  │  • Auto-scales during job execution                      │     │
│  │  • Pay only for DCUs (Data Compute Units) used           │     │
│  │  • Startup in ~60 seconds                                │     │
│  │                                                            │     │
│  │  Best for:                                                │     │
│  │  • Ad-hoc queries and exploration                        │     │
│  │  • Short-lived batch jobs                                │     │
│  │  • Teams without cluster management expertise            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Limitations:                                                       │
│  • PySpark and Spark SQL only (no Hadoop/Hive/Pig)               │
│  • No SSH access to workers                                       │
│  • Limited to supported Spark versions                            │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Submit Serverless PySpark job
gcloud dataproc batches submit pyspark \
    gs://my-bucket/scripts/etl.py \
    --region=us-central1 \
    --subnet=default \
    --jars=gs://spark-lib/bigquery/spark-bigquery-latest_2.12.jar \
    --properties=spark.executor.instances=10 \
    -- --input=gs://data/raw/ --output=gs://data/processed/

# Submit Serverless Spark SQL
gcloud dataproc batches submit spark-sql \
    --region=us-central1 \
    --file=gs://my-bucket/queries/report.sql

# List batches
gcloud dataproc batches list --region=us-central1

# Describe batch
gcloud dataproc batches describe BATCH_ID --region=us-central1

# Cancel batch
gcloud dataproc batches cancel BATCH_ID --region=us-central1
```

---

## Part 9 — Initialization Actions & Components

### Initialization Actions

```bash
# Run custom scripts at cluster creation
gcloud dataproc clusters create my-cluster \
    --region=us-central1 \
    --initialization-actions=\
gs://goog-dataproc-initialization-actions-us-central1/python/pip-install.sh,\
gs://my-bucket/scripts/custom-setup.sh \
    --metadata=PIP_PACKAGES='pandas scikit-learn nltk'
```

### Optional Components

```bash
# Available optional components:
# ANACONDA, DOCKER, DRUID, FLINK, HBASE, HIVE_WEBHCAT,
# JUPYTER, PRESTO, RANGER, SOLR, TRINO, ZEPPELIN, ZOOKEEPER

gcloud dataproc clusters create ml-cluster \
    --region=us-central1 \
    --optional-components=JUPYTER,DOCKER \
    --enable-component-gateway    # web UIs via Cloud IAP
```

---

## Part 10 — Metastore Service

### Dataproc Metastore (Managed Hive Metastore)

```
┌────────────────────────────────────────────────────────────────────┐
│         DATAPROC METASTORE                                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Shared metadata across clusters:                                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Dataproc Metastore (managed HMS)                         │     │
│  │  ┌──────────────────────────────────────┐                │     │
│  │  │ Database: analytics                   │                │     │
│  │  │ ├── Table: events (Parquet, GCS)     │                │     │
│  │  │ ├── Table: users (Parquet, GCS)      │                │     │
│  │  │ └── Table: sales (Parquet, GCS)      │                │     │
│  │  └──────────────────────────────────────┘                │     │
│  │       ▲            ▲            ▲                         │     │
│  │       │            │            │                         │     │
│  │  ┌────┴───┐  ┌─────┴────┐  ┌───┴──────┐                │     │
│  │  │Cluster │  │Cluster   │  │Serverless│                │     │
│  │  │  A     │  │  B       │  │  Batch   │                │     │
│  │  └────────┘  └──────────┘  └──────────┘                │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  • Persists table definitions across ephemeral clusters            │
│  • Shared catalog for Spark, Hive, Presto                         │
│  • Integrates with BigQuery (BigLake Metastore)                   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create metastore service
gcloud metastore services create my-metastore \
    --location=us-central1 \
    --tier=DEVELOPER

# Attach to cluster
gcloud dataproc clusters create my-cluster \
    --region=us-central1 \
    --dataproc-metastore=projects/my-project/locations/us-central1/services/my-metastore
```

---

## Part 11 — Persistent History Server

### Spark History Server

```bash
# Create persistent history server (PHS)
# Survives cluster deletion — view past job logs

gcloud dataproc clusters create history-server \
    --region=us-central1 \
    --single-node \
    --properties=\
spark:spark.history.fs.logDirectory=gs://my-bucket/spark-history/,\
spark:spark.eventLog.dir=gs://my-bucket/spark-history/,\
spark:spark.history.custom.executor.log.url={{YARN_LOG_SERVER_URL}}/{{NM_HOST}}:{{NM_PORT}}/{{CONTAINER_ID}}/{{CONTAINER_ID}}/{{USER}}/{{FILE_NAME}} \
    --enable-component-gateway

# Configure job clusters to write logs to same GCS path
gcloud dataproc clusters create job-cluster \
    --region=us-central1 \
    --properties=\
spark:spark.eventLog.enabled=true,\
spark:spark.eventLog.dir=gs://my-bucket/spark-history/
```

---

## Part 12 — Security & Networking

### Security Configuration

```bash
# Private cluster (no external IPs)
gcloud dataproc clusters create private-cluster \
    --region=us-central1 \
    --no-address \
    --subnet=my-private-subnet \
    --enable-component-gateway      # access UIs via IAP

# Kerberos-enabled cluster
gcloud dataproc clusters create kerb-cluster \
    --region=us-central1 \
    --enable-kerberos \
    --kerberos-root-principal-password-uri=\
gs://my-bucket/kerberos/password.encrypted \
    --kerberos-kms-key=projects/my-project/locations/global/keyRings/ring/cryptoKeys/key

# CMEK (Customer-Managed Encryption Keys)
gcloud dataproc clusters create encrypted-cluster \
    --region=us-central1 \
    --gce-pd-kms-key=projects/my-project/locations/us-central1/keyRings/ring/cryptoKeys/key

# Personal cluster authentication (per-user credentials)
gcloud dataproc clusters create multi-user \
    --region=us-central1 \
    --enable-component-gateway \
    --optional-components=JUPYTER \
    --properties=dataproc:dataproc.personal-auth.session.max-age-hours=48
```

---

## Part 13 — Monitoring & Troubleshooting

### Monitoring

```bash
# List jobs
gcloud dataproc jobs list --cluster=my-cluster --region=us-central1

# View job output
gcloud dataproc jobs wait JOB_ID --region=us-central1

# View job logs
gcloud logging read \
    'resource.type="cloud_dataproc_job" resource.labels.job_id="JOB_ID"' \
    --limit=50

# Key metrics (Cloud Monitoring):
# • dataproc.googleapis.com/cluster/yarn/yarn_memory_size
# • dataproc.googleapis.com/cluster/yarn/yarn_virtual_cores
# • dataproc.googleapis.com/cluster/hdfs/storage_utilization
# • dataproc.googleapis.com/cluster/job/duration
```

### Web UIs (Component Gateway)

| UI | Purpose |
|----|---------|
| YARN ResourceManager | Job/container status |
| Spark History Server | Spark job details, DAGs, stages |
| Jupyter / JupyterLab | Interactive notebooks |
| HDFS NameNode | HDFS file browser |
| Tez UI | Hive query visualization |

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Autoscaling Policy ──────────────────────────────────────
resource "google_dataproc_autoscaling_policy" "default" {
  policy_id = "dataproc-policy"
  location  = var.region
  project   = var.project_id

  worker_config {
    min_instances = 2
    max_instances = 10
    weight        = 1
  }

  secondary_worker_config {
    min_instances = 0
    max_instances = 50
    weight        = 1
  }

  basic_algorithm {
    yarn_config {
      scale_up_factor               = 1.0
      scale_down_factor             = 1.0
      scale_up_min_worker_fraction  = 0.0
      cooldown_period               = "120s"
      graceful_decommission_timeout = "3600s"
    }
  }
}

# ─── Dataproc Cluster ────────────────────────────────────────
resource "google_dataproc_cluster" "primary" {
  name    = "analytics-cluster"
  region  = var.region
  project = var.project_id

  cluster_config {
    staging_bucket = google_storage_bucket.staging.name
    temp_bucket    = google_storage_bucket.temp.name

    master_config {
      num_instances = 1
      machine_type  = "n2-standard-4"
      disk_config {
        boot_disk_type    = "pd-ssd"
        boot_disk_size_gb = 100
      }
    }

    worker_config {
      num_instances = 3
      machine_type  = "n2-standard-8"
      disk_config {
        boot_disk_type    = "pd-standard"
        boot_disk_size_gb = 200
        num_local_ssds    = 1
      }
    }

    preemptible_worker_config {
      num_instances = 5
      preemptibility = "SPOT"
      disk_config {
        boot_disk_type    = "pd-standard"
        boot_disk_size_gb = 100
      }
    }

    software_config {
      image_version = "2.1-debian11"
      override_properties = {
        "spark:spark.executor.memory"      = "4g"
        "spark:spark.driver.memory"        = "2g"
        "spark:spark.eventLog.enabled"     = "true"
        "spark:spark.eventLog.dir"         = "gs://${google_storage_bucket.staging.name}/spark-history/"
      }
      optional_components = ["JUPYTER", "DOCKER"]
    }

    gce_cluster_config {
      subnetwork       = google_compute_subnetwork.dataproc.self_link
      internal_ip_only = true
      service_account  = google_service_account.dataproc.email
      service_account_scopes = ["cloud-platform"]

      shielded_instance_config {
        enable_secure_boot = true
      }
    }

    autoscaling_config {
      policy_uri = google_dataproc_autoscaling_policy.default.id
    }

    endpoint_config {
      enable_http_port_access = true    # component gateway
    }
  }
}

# ─── Dataproc Job ─────────────────────────────────────────────
resource "google_dataproc_job" "etl" {
  region = var.region
  project = var.project_id

  placement {
    cluster_name = google_dataproc_cluster.primary.name
  }

  pyspark_config {
    main_python_file_uri = "gs://${google_storage_bucket.scripts.name}/etl.py"
    jar_file_uris = [
      "gs://spark-lib/bigquery/spark-bigquery-with-dependencies_2.12-0.36.1.jar"
    ]
    properties = {
      "spark.executor.memory" = "8g"
    }
    args = ["--date", "2024-01-15"]
  }
}

# ─── Serverless Batch ─────────────────────────────────────────
resource "google_dataproc_batch" "serverless" {
  batch_id = "etl-batch-001"
  location = var.region
  project  = var.project_id

  pyspark_batch {
    main_python_file_uri = "gs://${google_storage_bucket.scripts.name}/etl.py"
    jar_file_uris = [
      "gs://spark-lib/bigquery/spark-bigquery-with-dependencies_2.12-0.36.1.jar"
    ]
    args = ["--date", "2024-01-15"]
  }

  runtime_config {
    version = "2.1"
    properties = {
      "spark.executor.instances" = "10"
    }
  }

  environment_config {
    execution_config {
      subnetwork_uri = google_compute_subnetwork.dataproc.id
      service_account = google_service_account.dataproc.email
    }
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# CLUSTERS
# ═══════════════════════════════════════════════════════════════
gcloud dataproc clusters create NAME --region=R [opts]
gcloud dataproc clusters update NAME --region=R [opts]
gcloud dataproc clusters describe NAME --region=R
gcloud dataproc clusters list --region=R
gcloud dataproc clusters delete NAME --region=R

# ═══════════════════════════════════════════════════════════════
# JOBS
# ═══════════════════════════════════════════════════════════════
gcloud dataproc jobs submit pyspark FILE --cluster=C --region=R [opts]
gcloud dataproc jobs submit spark --cluster=C --region=R --class=CLS [opts]
gcloud dataproc jobs submit hive --cluster=C --region=R --file=F
gcloud dataproc jobs submit spark-sql --cluster=C --region=R --file=F
gcloud dataproc jobs list --cluster=C --region=R
gcloud dataproc jobs describe JOB_ID --region=R
gcloud dataproc jobs kill JOB_ID --region=R

# ═══════════════════════════════════════════════════════════════
# SERVERLESS BATCHES
# ═══════════════════════════════════════════════════════════════
gcloud dataproc batches submit pyspark FILE --region=R [opts]
gcloud dataproc batches submit spark-sql --region=R --file=F
gcloud dataproc batches list --region=R
gcloud dataproc batches describe BATCH_ID --region=R
gcloud dataproc batches cancel BATCH_ID --region=R

# ═══════════════════════════════════════════════════════════════
# AUTOSCALING
# ═══════════════════════════════════════════════════════════════
gcloud dataproc autoscaling-policies create NAME --region=R [opts]
gcloud dataproc autoscaling-policies list --region=R
gcloud dataproc autoscaling-policies describe NAME --region=R
gcloud dataproc autoscaling-policies delete NAME --region=R
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Ephemeral Cluster Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: EPHEMERAL CLUSTER (CREATE → RUN → DELETE)             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Cloud Composer (Airflow DAG)                              │        │
│  │                                                            │        │
│  │  Task 1: create_cluster                                   │        │
│  │  → gcloud dataproc clusters create ephemeral-cluster      │        │
│  │    (autoscaling, Spot workers, 2 primary + 10 secondary) │        │
│  │                                                            │        │
│  │  Task 2: submit_etl_job                                   │        │
│  │  → gcloud dataproc jobs submit pyspark etl.py             │        │
│  │    (reads GCS parquet, transforms, writes to BigQuery)    │        │
│  │                                                            │        │
│  │  Task 3: submit_aggregation_job                           │        │
│  │  → gcloud dataproc jobs submit spark-sql aggregate.sql   │        │
│  │                                                            │        │
│  │  Task 4: delete_cluster                                   │        │
│  │  → gcloud dataproc clusters delete ephemeral-cluster      │        │
│  │    (always runs, even if jobs fail)                       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Cost: pay only for job duration (~30 min/day)                      │
│  Data: persists in GCS + BigQuery (not on cluster HDFS)             │
│  Metadata: Dataproc Metastore (shared across clusters)              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Migration from On-Prem Hadoop

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: HADOOP MIGRATION                                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Phase 1: Lift & Shift                                               │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ • Move HDFS data to GCS (distcp or Transfer Service)    │        │
│  │ • Create Dataproc cluster matching on-prem config       │        │
│  │ • Run same Spark/Hive jobs with minimal changes         │        │
│  │ • Replace hdfs:// paths with gs:// paths                │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 2: Optimize                                                   │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ • Switch to ephemeral clusters (create → run → delete)  │        │
│  │ • Enable autoscaling + Spot workers                     │        │
│  │ • Use Dataproc Serverless for ad-hoc queries            │        │
│  │ • Migrate Hive tables to BigQuery where possible        │        │
│  │ • Set up Dataproc Metastore (shared metadata)           │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Phase 3: Modernize                                                  │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ • Move batch ETL to Dataflow (unified batch/stream)     │        │
│  │ • Move analytics queries to BigQuery SQL                │        │
│  │ • Keep Spark for ML workloads (MLlib, Spark ML)         │        │
│  │ • Orchestrate with Cloud Composer (Airflow)             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Cost-Optimized ML Training

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: ML TRAINING ON DATAPROC                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Dataproc cluster for ML:                                  │        │
│  │                                                            │        │
│  │  Master: n2-standard-4 (1 node)                          │        │
│  │  Primary workers: n2-standard-8 (2 nodes, on-demand)    │        │
│  │  Secondary workers: n2-standard-8 (20 Spot VMs)         │        │
│  │  Components: Jupyter, Anaconda                            │        │
│  │                                                            │        │
│  │  Pipeline:                                                │        │
│  │  1. Data prep: Spark reads TB-scale data from GCS       │        │
│  │  2. Feature eng: Spark MLlib transformations             │        │
│  │  3. Training: Spark MLlib / XGBoost distributed          │        │
│  │  4. Evaluation: metrics to Cloud Monitoring              │        │
│  │  5. Model export: save to GCS for Vertex AI serving     │        │
│  │                                                            │        │
│  │  Cost savings:                                            │        │
│  │  • Spot VMs: ~70% discount on 20 workers                │        │
│  │  • Ephemeral: cluster exists only during training        │        │
│  │  • Checkpoint to GCS: resume if Spot reclaimed           │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create cluster | `gcloud dataproc clusters create C --region=R [opts]` |
| Delete cluster | `gcloud dataproc clusters delete C --region=R` |
| Submit PySpark | `gcloud dataproc jobs submit pyspark FILE --cluster=C --region=R` |
| Submit Hive | `gcloud dataproc jobs submit hive --file=F --cluster=C --region=R` |
| Serverless batch | `gcloud dataproc batches submit pyspark FILE --region=R` |
| Enable autoscaling | `--autoscaling-policy=POLICY` |
| Spot workers | `--num-secondary-workers=N --secondary-worker-type=spot` |
| Component gateway | `--enable-component-gateway` |
| HA cluster | `--num-masters=3` |
| Private cluster | `--no-address --subnet=S` |
| Pricing | VM cost + $0.01/vCPU-hr Dataproc premium |

---

## What is Spark/Hadoop? (Beginner Explanation)

If you've never worked with Spark or Hadoop, here's the simplest way to understand them:

```
┌────────────────────────────────────────────────────────────────────┐
│         SPARK — THE BIG IDEA                                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Imagine you have a 10,000-piece puzzle to solve.                  │
│                                                                      │
│  Without Spark (1 person):                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  One person sits alone and assembles all 10,000 pieces.  │     │
│  │  Takes: a very long time.                                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  With Spark (100 people):                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  You split the puzzle into 100 sections and give each    │     │
│  │  section to a different person. They all work at the     │     │
│  │  same time (in parallel). Then you combine their work.   │     │
│  │  Takes: a fraction of the time.                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  That's distributed computing!                                     │
│  • Spark = the coordinator that splits work + combines results    │
│  • Workers = the people doing the actual work (VMs in the cloud) │
│  • Data = the puzzle pieces (stored in GCS or BigQuery)           │
│                                                                      │
│  Hadoop was the original framework for this (older, slower).      │
│  Spark replaced most Hadoop workloads because it runs             │
│  10–100x faster by keeping data in memory instead of on disk.    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### When to Use Dataproc vs Dataflow

| Question | Dataproc | Dataflow |
|----------|----------|----------|
| What engine? | Apache Spark / Hadoop / Hive | Apache Beam |
| Best for? | Existing Spark/Hadoop jobs, ML with MLlib, Hive SQL | New pipelines, streaming, unified batch+stream |
| Cluster management? | You manage clusters (or use Serverless) | Fully serverless — no clusters ever |
| Languages? | Python, Scala, Java, SQL, R | Python, Java, Go |
| Streaming? | Spark Structured Streaming (good) | Native streaming with exactly-once (excellent) |
| Migrating from on-prem Hadoop? | Yes — drop-in replacement | No — requires rewriting in Beam |
| Interactive notebooks? | Yes — Jupyter built-in | No |
| Cost model? | VM cost + $0.01/vCPU-hr premium | Per vCPU-hr + memory + shuffle |

> **Rule of thumb:**
> - Already have Spark/Hadoop jobs? → **Dataproc**
> - Starting fresh or need real-time streaming? → **Dataflow**
> - Need Hive SQL on existing data? → **Dataproc**
> - Need exactly-once streaming from Pub/Sub? → **Dataflow**

---

## Console Walkthrough: Creating Clusters & Running Jobs

### Creating a Cluster from Console

```
Console → Dataproc → Clusters → CREATE CLUSTER

┌─────────────────────────────────────────────────────────────────┐
│           STEP-BY-STEP: CREATE A DATAPROC CLUSTER                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Navigate                                                 │
│ ─────────────────                                                │
│ Console → search "Dataproc" → Click "Dataproc" → "Clusters"   │
│ → Click "CREATE CLUSTER" at the top                             │
│ → Select "Cluster on Compute Engine"                            │
│                                                                   │
│ Step 2: Cluster Name & Location                                 │
│ ────────────────────────────                                     │
│ • Name: [my-spark-cluster]                                      │
│   (lowercase letters, numbers, hyphens)                         │
│ • Region: [us-central1 ▼]                                       │
│ • Zone: [us-central1-a ▼] (or "Any" for auto-zone)             │
│ • Cluster type:                                                  │
│   ● Standard (1 master)                                        │
│   ○ High Availability (3 masters)                              │
│   ○ Single Node (master only, for dev/testing)                 │
│                                                                   │
│ Step 3: Configure Nodes                                          │
│ ───────────────────                                              │
│                                                                   │
│ ── Master Node ──                                                │
│ • Machine type: [n2-standard-4 ▼] (4 vCPU, 16 GB)              │
│ • Primary disk size: [100 GB]                                   │
│ • Primary disk type: [Standard ▼]                               │
│                                                                   │
│ ── Primary Worker Nodes ──                                      │
│ • Number of workers: [3]                                        │
│ • Machine type: [n2-standard-4 ▼]                               │
│ • Primary disk size: [200 GB]                                   │
│                                                                   │
│ ── Secondary Worker Nodes (optional) ──                         │
│ • Number: [0] (add for extra compute at lower cost)             │
│ • Preemptibility: [Spot ▼] (60-91% cheaper, can be reclaimed)  │
│ • These workers have NO local HDFS — compute only               │
│                                                                   │
│ Step 4: Customize Cluster (expand panel)                        │
│ ────────────────────────────────────                             │
│ • Image version: [2.2 ▼] (determines Spark/Hadoop versions)    │
│ • Optional components: ☑ Jupyter  ☑ Docker  ☐ Presto           │
│ • ☑ Enable component gateway (access web UIs securely)         │
│ • Scheduled deletion: ☐ (auto-delete after N hours of idle)    │
│ • Autoscaling policy: [None ▼] or select an existing policy    │
│                                                                   │
│ Step 5: Manage Security (expand panel)                          │
│ ──────────────────────────────────                               │
│ • Service account: [default or custom SA]                       │
│ • Network/Subnetwork: [default ▼]                               │
│ • ☐ Internal IP only (private cluster)                          │
│ • Encryption: [Google-managed ▼] or CMEK                       │
│                                                                   │
│ Step 6: Click [CREATE]                                          │
│ → Cluster provisions in ~90 seconds                             │
│ → Status changes from "Provisioning" → "Running"               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Submitting a Job from Console

```
Console → Dataproc → Jobs → SUBMIT JOB

┌─────────────────────────────────────────────────────────────────┐
│           STEP-BY-STEP: SUBMIT A JOB                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Navigate                                                 │
│ ─────────────────                                                │
│ Console → Dataproc → Jobs → Click "SUBMIT JOB"                 │
│                                                                   │
│ Step 2: Select Cluster & Job Type                               │
│ ─────────────────────────────                                    │
│ • Region: [us-central1 ▼]                                       │
│ • Cluster: [my-spark-cluster ▼]                                 │
│ • Job type: [PySpark ▼]                                         │
│   Options: PySpark, Spark, SparkR, Hive, SparkSQL,             │
│            Pig, Hadoop, Presto, Trino                           │
│                                                                   │
│ Step 3: Job Details (vary by job type)                          │
│ ──────────────────────────────────                               │
│ For PySpark:                                                     │
│ • Main Python file: [gs://my-bucket/scripts/etl.py]            │
│ • Arguments: [--input gs://data/ --output gs://results/]       │
│ • Jar files: [gs://spark-lib/bigquery/spark-bigquery.jar]      │
│ • Python files: (additional .py modules)                        │
│ • Properties: e.g. spark.executor.memory=4g                    │
│ • Max restarts per hour: [0]                                    │
│                                                                   │
│ Step 4: Click [SUBMIT]                                          │
│ → Job appears in the jobs list with status "Running"            │
│ → Click job ID to view details and output                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Viewing Job Output and Logs

```
Console → Dataproc → Jobs → Click on your Job ID

┌─────────────────────────────────────────────────────────────────┐
│           VIEWING JOB OUTPUT & LOGS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Job Details Page ──                                          │
│ Status: ● Running  |  Submitted: 2 minutes ago                  │
│ Cluster: my-spark-cluster  |  Type: PySpark                    │
│                                                                   │
│ ── Output Tab (default) ──                                      │
│ Shows stdout/stderr from your job in real time:                 │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ 24/01/15 14:02:03 INFO SparkContext: Running Spark 3.5  │    │
│ │ Processing 15,234 records...                              │    │
│ │ Region: us-east1 — 5,120 records                          │    │
│ │ Region: eu-west1 — 10,114 records                         │    │
│ │ Job completed successfully.                                │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ ── Configuration Tab ──                                         │
│ Shows all job parameters, Spark properties, and JAR files.     │
│                                                                   │
│ ── Monitoring Tab ──                                            │
│ Links to:                                                       │
│ • YARN ResourceManager (job/container status)                  │
│ • Spark History Server (stages, DAG, executors)                │
│ • Cloud Logging (click "View Logs" for full log exploration)  │
│                                                                   │
│ 💡 Tip: Enable Component Gateway when creating the cluster     │
│    to access Spark UI, YARN, and Jupyter through the Console.  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deleting a Cluster from Console

```
Console → Dataproc → Clusters

┌─────────────────────────────────────────────────────────────────┐
│           DELETING A CLUSTER                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Option A: From Cluster List                                     │
│ ────────────────────────                                         │
│ 1. Console → Dataproc → Clusters                                │
│ 2. Check the box ☑ next to your cluster name                   │
│ 3. Click "DELETE" at the top                                    │
│ 4. Confirm in the dialog                                        │
│                                                                   │
│ Option B: From Cluster Details                                  │
│ ──────────────────────────                                       │
│ 1. Click on the cluster name to open details                   │
│ 2. Click "DELETE" at the top of the page                       │
│ 3. Confirm in the dialog                                        │
│                                                                   │
│ ── What Happens ──                                              │
│ • All worker and master VMs are terminated                     │
│ • Local HDFS data on the cluster is permanently deleted        │
│ • Data in GCS is NOT deleted (survives cluster deletion)       │
│ • Dataproc Metastore tables are NOT affected                   │
│ • Billing for VMs + Dataproc premium stops immediately         │
│                                                                   │
│ ── Best Practice ──                                             │
│ Use ephemeral clusters: Create → Run Job → Delete              │
│ Store all data in GCS (not HDFS) so nothing is lost            │
│ when the cluster is deleted.                                    │
│                                                                   │
│ 💡 Tip: Enable "Scheduled deletion" when creating the cluster  │
│    to auto-delete after a period of inactivity. This prevents  │
│    accidentally leaving clusters running and wasting money.     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 57: Cloud Composer (Airflow)** → `57-cloud-composer.md`
