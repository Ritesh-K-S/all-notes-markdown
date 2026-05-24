# Chapter 61 — Storage Transfer & BigQuery Transfer

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Data Transfer Fundamentals](#part-1--data-transfer-fundamentals)
- [Part 2: Storage Transfer Service — Basics](#part-2--storage-transfer-service--basics)
- [Part 3: Storage Transfer Service — Cloud-to-Cloud](#part-3--storage-transfer-service--cloud-to-cloud)
- [Part 4: Storage Transfer Service — On-Premises Agent](#part-4--storage-transfer-service--on-premises-agent)
- [Part 5: Storage Transfer Service — POSIX Filesystems](#part-5--storage-transfer-service--posix-filesystems)
- [Part 6: Storage Transfer Service — Advanced Options](#part-6--storage-transfer-service--advanced-options)
- [Part 7: gsutil & gcloud storage for Transfers](#part-7--gsutil--gcloud-storage-for-transfers)
- [Part 8: BigQuery Data Transfer Service — Basics](#part-8--bigquery-data-transfer-service--basics)
- [Part 9: BigQuery Data Transfer — SaaS Sources](#part-9--bigquery-data-transfer--saas-sources)
- [Part 10: BigQuery Data Transfer — Cross-Region & Dataset Copy](#part-10--bigquery-data-transfer--cross-region--dataset-copy)
- [Part 11: BigQuery Data Loading (Non-Transfer)](#part-11--bigquery-data-loading-non-transfer)
- [Part 12: Transfer Appliance (Physical)](#part-12--transfer-appliance-physical)
- [Part 13: Choosing the Right Transfer Method](#part-13--choosing-the-right-transfer-method)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [When to Use Which Transfer Method? (Beginner Guide)](#when-to-use-which-transfer-method-beginner-guide)
- [Console Walkthrough: Creating Transfer Jobs](#console-walkthrough-creating-transfer-jobs)
- [What's Next?](#whats-next)

---

## Overview

Moving data into, out of, and across Google Cloud is a critical part of any cloud strategy. GCP provides multiple transfer services depending on the data volume, source, and frequency: Storage Transfer Service for cloud-to-cloud and on-prem-to-cloud object transfers, BigQuery Data Transfer Service for automated data ingestion into BigQuery from SaaS sources and other clouds, and Transfer Appliance for physical bulk data shipping. This chapter covers every transfer option, when to use each, and how to configure them.

---

## Part 1 — Data Transfer Fundamentals

### Transfer Method Decision Tree

```
┌────────────────────────────────────────────────────────────────────┐
│         CHOOSING A TRANSFER METHOD                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  What are you transferring?                                        │
│       │                                                             │
│       ├── Objects/files → Cloud Storage (GCS)                     │
│       │       │                                                     │
│       │       ├── < 1 TB?                                         │
│       │       │    └── gsutil / gcloud storage cp                 │
│       │       │                                                     │
│       │       ├── 1 TB – 20 TB?                                   │
│       │       │    └── Storage Transfer Service                   │
│       │       │                                                     │
│       │       ├── > 20 TB & slow network?                         │
│       │       │    └── Transfer Appliance                         │
│       │       │                                                     │
│       │       ├── From AWS S3 / Azure Blob?                       │
│       │       │    └── Storage Transfer Service                   │
│       │       │                                                     │
│       │       └── Recurring / scheduled?                          │
│       │            └── Storage Transfer Service                   │
│       │                                                             │
│       ├── Structured data → BigQuery                              │
│       │       │                                                     │
│       │       ├── From SaaS (Google Ads, YouTube, etc.)?         │
│       │       │    └── BigQuery Data Transfer Service             │
│       │       │                                                     │
│       │       ├── From S3, GCS, or other dataset?                │
│       │       │    └── BigQuery Data Transfer Service             │
│       │       │                                                     │
│       │       ├── One-time load from GCS?                        │
│       │       │    └── bq load / BigQuery console                │
│       │       │                                                     │
│       │       └── Real-time streaming?                           │
│       │            └── Pub/Sub + Dataflow + BigQuery             │
│       │                                                             │
│       └── Database → managed DB                                   │
│            └── Database Migration Service (Ch 60)                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP | AWS | Azure |
|---------|-----|-----|-------|
| Cloud-to-cloud transfer | Storage Transfer Service | DataSync | Data Factory |
| On-prem file transfer | Storage Transfer Agent | DataSync Agent | Data Box Gateway |
| Physical appliance | Transfer Appliance | Snowball / Snowmobile | Data Box |
| Data warehouse ingestion | BigQuery Data Transfer | Glue / AppFlow | Data Factory |
| CLI transfer | gcloud storage cp / gsutil | aws s3 cp | azcopy |
| SaaS connectors | BQDTS (Ads, YouTube, etc.) | AppFlow | Data Factory |

---

## Part 2 — Storage Transfer Service — Basics

### Service Overview

```
┌────────────────────────────────────────────────────────────────────┐
│         STORAGE TRANSFER SERVICE                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Managed service for transferring data into GCS:                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Sources supported:                                       │     │
│  │  ┌────────────────────────────────────────────────┐      │     │
│  │  │ • Amazon S3 buckets                             │      │     │
│  │  │ • Azure Blob Storage                            │      │     │
│  │  │ • Other GCS buckets (cross-region/project)     │      │     │
│  │  │ • HTTP/HTTPS URL lists (public files)          │      │     │
│  │  │ • On-premises file systems (via agent)         │      │     │
│  │  │ • POSIX-compatible file systems                │      │     │
│  │  │ • HDFS (Hadoop Distributed File System)        │      │     │
│  │  └────────────────────────────────────────────────┘      │     │
│  │                                                            │     │
│  │  Destination: GCS bucket                                  │     │
│  │                                                            │     │
│  │  Features:                                                │     │
│  │  • Scheduled / recurring transfers                       │     │
│  │  • Include/exclude filters (prefixes, modified time)    │     │
│  │  • Delete source or destination objects after transfer   │     │
│  │  • Bandwidth limits                                      │     │
│  │  • Manifest file support                                 │     │
│  │  • Logging & monitoring                                  │     │
│  │  • Retry on failure                                      │     │
│  │                                                            │     │
│  │  Pricing: FREE (no charge for the service itself)        │     │
│  │  You pay only for egress from source cloud + GCS storage │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 3 — Storage Transfer Service — Cloud-to-Cloud

### Transfer from AWS S3 / Azure Blob / GCS

```bash
# ═══════════════════════════════════════════════════════════════
# TRANSFER FROM AWS S3 TO GCS
# ═══════════════════════════════════════════════════════════════
gcloud transfer jobs create \
    s3://source-bucket \
    gs://destination-bucket \
    --source-creds-file=aws_credentials.json \
    --name=s3-to-gcs-daily \
    --description="Daily sync from S3" \
    --schedule-repeats-every=24h \
    --schedule-starts=2024-01-15T00:00:00Z

# aws_credentials.json format:
# {
#   "accessKeyId": "AKIA...",
#   "secretAccessKey": "..."
# }

# ═══════════════════════════════════════════════════════════════
# TRANSFER FROM AZURE BLOB TO GCS
# ═══════════════════════════════════════════════════════════════
gcloud transfer jobs create \
    https://myaccount.blob.core.windows.net/container \
    gs://destination-bucket \
    --source-creds-file=azure_credentials.json \
    --name=azure-to-gcs

# azure_credentials.json format:
# {
#   "sasToken": "sv=2021-..."
# }

# ═══════════════════════════════════════════════════════════════
# TRANSFER BETWEEN GCS BUCKETS (cross-region/project)
# ═══════════════════════════════════════════════════════════════
gcloud transfer jobs create \
    gs://source-bucket \
    gs://destination-bucket \
    --name=gcs-cross-region \
    --include-prefixes=data/2024/ \
    --exclude-prefixes=data/2024/temp/

# ═══════════════════════════════════════════════════════════════
# TRANSFER FROM URL LIST
# ═══════════════════════════════════════════════════════════════
# Create a TSV file with URLs:
# TsvHttpData-1.0
# https://example.com/file1.zip    1234567    md5hash
# https://example.com/file2.zip    2345678    md5hash

gcloud transfer jobs create \
    https://example.com/url_list.tsv \
    gs://destination-bucket \
    --name=url-download
```

---

## Part 4 — Storage Transfer Service — On-Premises Agent

### Agent-Based Transfer (On-Prem → GCS)

```
┌────────────────────────────────────────────────────────────────────┐
│         ON-PREMISES TRANSFER                                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────┐         ┌──────────────────┐                │
│  │  On-Premises      │         │  Google Cloud     │                │
│  │                    │         │                    │                │
│  │  ┌──────────────┐ │         │  ┌──────────────┐ │                │
│  │  │ File Server  │ │         │  │ GCS Bucket   │ │                │
│  │  │ /data/       │ │         │  │              │ │                │
│  │  └──────┬───────┘ │         │  └──────▲───────┘ │                │
│  │         │          │         │         │          │                │
│  │  ┌──────▼───────┐ │         │         │          │                │
│  │  │ Transfer     │ │ ──TLS──►│         │          │                │
│  │  │ Agent(s)     │ │         │  Storage Transfer  │                │
│  │  │ (Docker)     │ │         │  Service            │                │
│  │  └──────────────┘ │         │  (orchestrates)    │                │
│  └──────────────────┘         └──────────────────┘                │
│                                                                      │
│  Multiple agents = parallel transfer (higher throughput)          │
│  Recommended: 3+ agents for large transfers                       │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Step 1: Create an agent pool
gcloud transfer agent-pools create my-agent-pool \
    --display-name="On-Prem Agent Pool"

# Step 2: Install and run transfer agents (Docker)
# Run on each on-prem machine:
docker run -d \
    --name=transfer-agent \
    --ulimit memlock=64000000 \
    -v /data:/data:ro \
    gcr.io/cloud-ingest/tsop-agent:latest \
    --project-id=my-project \
    --agent-pool=my-agent-pool \
    --hostname=$(hostname) \
    --log-dir=/tmp/agent-logs

# Step 3: Create transfer job
gcloud transfer jobs create \
    posix:///data/ \
    gs://destination-bucket \
    --source-agent-pool=my-agent-pool \
    --name=onprem-to-gcs \
    --schedule-repeats-every=6h
```

---

## Part 5 — Storage Transfer Service — POSIX Filesystems

### Transfer Between POSIX and GCS

```bash
# Transfer from POSIX filesystem (e.g., NFS, local disk) to GCS
gcloud transfer jobs create \
    posix:///mnt/nfs/shared/ \
    gs://destination-bucket/nfs-backup/ \
    --source-agent-pool=my-pool \
    --name=nfs-backup-daily \
    --schedule-repeats-every=24h

# Transfer from GCS to POSIX filesystem
gcloud transfer jobs create \
    gs://source-bucket/data/ \
    posix:///mnt/local/restore/ \
    --destination-agent-pool=my-pool \
    --name=gcs-to-local

# Transfer between two POSIX filesystems (via agents)
gcloud transfer jobs create \
    posix:///mnt/source/ \
    posix:///mnt/destination/ \
    --source-agent-pool=source-pool \
    --destination-agent-pool=dest-pool \
    --name=posix-to-posix

# HDFS to GCS
gcloud transfer jobs create \
    hdfs:///data/warehouse/ \
    gs://destination-bucket/hdfs-data/ \
    --source-agent-pool=hadoop-pool \
    --name=hdfs-to-gcs
```

---

## Part 6 — Storage Transfer Service — Advanced Options

### Filters, Scheduling, and Options

```bash
# Include/exclude filters
gcloud transfer jobs create \
    gs://source-bucket \
    gs://dest-bucket \
    --include-prefixes=logs/2024/,data/production/ \
    --exclude-prefixes=logs/2024/debug/,data/production/temp/ \
    --include-modified-since=2024-01-01T00:00:00Z \
    --include-modified-before=2024-12-31T23:59:59Z

# Delete options
gcloud transfer jobs create \
    gs://source-bucket \
    gs://dest-bucket \
    --delete-from=source-after-transfer     # mirror mode
    # Options: source-after-transfer, destination-if-unique

# Overwrite options
gcloud transfer jobs create \
    gs://source-bucket \
    gs://dest-bucket \
    --overwrite-when=different              # only if different
    # Options: always, different, never

# Bandwidth limit
gcloud transfer jobs create \
    s3://source \
    gs://dest \
    --source-creds-file=creds.json \
    --bandwidth-limit=100MB                 # limit to 100 MB/s

# Manifest file (specific files only)
gcloud transfer jobs create \
    gs://source-bucket \
    gs://dest-bucket \
    --manifest-file=gs://config-bucket/manifest.csv
    # manifest.csv: one object path per line

# Job management
gcloud transfer jobs list
gcloud transfer jobs describe JOB_NAME
gcloud transfer jobs run JOB_NAME            # trigger immediately
gcloud transfer jobs delete JOB_NAME
gcloud transfer operations list --job-names=JOB_NAME
gcloud transfer operations describe OP_NAME
```

---

## Part 7 — gsutil & gcloud storage for Transfers

### Command-Line Data Transfer

```bash
# ═══════════════════════════════════════════════════════════════
# gcloud storage (recommended — newer, faster)
# ═══════════════════════════════════════════════════════════════

# Copy single file
gcloud storage cp local-file.csv gs://my-bucket/data/

# Copy directory recursively
gcloud storage cp -r ./my-data/ gs://my-bucket/data/

# Download from GCS
gcloud storage cp gs://my-bucket/data/file.csv ./local/

# Sync directory (like rsync)
gcloud storage rsync -r ./local-dir/ gs://my-bucket/backup/
gcloud storage rsync -r -d gs://my-bucket/backup/ ./local-dir/  # -d = delete extra

# Parallel composite upload (large files)
gcloud storage cp --component-size=50M large-file.tar.gz gs://my-bucket/

# ═══════════════════════════════════════════════════════════════
# gsutil (legacy — still widely used)
# ═══════════════════════════════════════════════════════════════

# Copy with parallel threads
gsutil -m cp -r ./data/ gs://my-bucket/

# Sync
gsutil -m rsync -r ./local/ gs://my-bucket/backup/

# Parallel composite upload
gsutil -o GSUtil:parallel_composite_upload_threshold=150M \
    cp large-file.tar.gz gs://my-bucket/

# Set parallel process/thread counts
gsutil -o "GSUtil:parallel_process_count=8" \
    -o "GSUtil:parallel_thread_count=4" \
    -m cp -r ./data/ gs://my-bucket/
```

### When to Use Which

| Method | Best For | Max Throughput |
|--------|---------|---------------|
| `gcloud storage cp` | Ad-hoc transfers < 1 TB | Limited by bandwidth |
| `gsutil -m` | Parallel transfers < 1 TB | Limited by bandwidth |
| Storage Transfer Service | > 1 TB, scheduled, cross-cloud | Managed (auto-scaled) |
| Transfer Appliance | > 20 TB, slow network | Physical shipping speed |

---

## Part 8 — BigQuery Data Transfer Service — Basics

### Automated Data Ingestion

```
┌────────────────────────────────────────────────────────────────────┐
│         BIGQUERY DATA TRANSFER SERVICE                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Automated, scheduled data loading into BigQuery:                 │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Supported sources:                                       │     │
│  │                                                            │     │
│  │  Google SaaS:                                             │     │
│  │  ├── Google Ads                                           │     │
│  │  ├── Google Ad Manager                                   │     │
│  │  ├── Google Play                                          │     │
│  │  ├── YouTube Channel / Content Owner                     │     │
│  │  ├── Google Merchant Center                              │     │
│  │  ├── Campaign Manager                                    │     │
│  │  └── Search Ads 360                                      │     │
│  │                                                            │     │
│  │  Cloud sources:                                           │     │
│  │  ├── Amazon S3                                           │     │
│  │  ├── Amazon Redshift                                     │     │
│  │  ├── Azure Blob Storage                                  │     │
│  │  ├── Teradata                                            │     │
│  │  └── Google Cloud Storage (scheduled loads)             │     │
│  │                                                            │     │
│  │  Cross-region:                                            │     │
│  │  └── BigQuery dataset copy (region-to-region)           │     │
│  │                                                            │     │
│  │  Third-party (via partners):                             │     │
│  │  ├── Salesforce                                          │     │
│  │  ├── Facebook Ads                                        │     │
│  │  ├── LinkedIn Ads                                        │     │
│  │  ├── Oracle                                              │     │
│  │  └── SAP                                                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pricing: FREE (the transfer service itself)                      │
│  You pay for BigQuery storage + queries                           │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 9 — BigQuery Data Transfer — SaaS Sources

### Configuring Transfers

```bash
# ═══════════════════════════════════════════════════════════════
# GOOGLE ADS TRANSFER
# ═══════════════════════════════════════════════════════════════
bq mk --transfer_config \
    --data_source=google_ads \
    --display_name="Google Ads Daily" \
    --target_dataset=ads_data \
    --schedule="every 24 hours" \
    --params='{
        "customer_id": "123-456-7890"
    }'

# ═══════════════════════════════════════════════════════════════
# YOUTUBE CHANNEL TRANSFER
# ═══════════════════════════════════════════════════════════════
bq mk --transfer_config \
    --data_source=youtube_channel \
    --display_name="YouTube Analytics" \
    --target_dataset=youtube_data \
    --schedule="every 24 hours" \
    --params='{
        "table_suffix": "_daily"
    }'

# ═══════════════════════════════════════════════════════════════
# AMAZON S3 TRANSFER → BIGQUERY
# ═══════════════════════════════════════════════════════════════
bq mk --transfer_config \
    --data_source=amazon_s3 \
    --display_name="S3 Daily Import" \
    --target_dataset=imported_data \
    --schedule="every 24 hours" \
    --params='{
        "destination_table_name_template": "s3_data_{run_date}",
        "data_path": "s3://my-bucket/data/*.csv",
        "access_key_id": "AKIA...",
        "secret_access_key": "...",
        "file_format": "CSV",
        "skip_leading_rows": "1",
        "write_disposition": "APPEND"
    }'

# ═══════════════════════════════════════════════════════════════
# SCHEDULED GCS LOAD → BIGQUERY
# ═══════════════════════════════════════════════════════════════
bq mk --transfer_config \
    --data_source=google_cloud_storage \
    --display_name="GCS Daily Load" \
    --target_dataset=analytics \
    --schedule="every 24 hours" \
    --params='{
        "destination_table_name_template": "events_{run_date}",
        "data_path_template": "gs://my-bucket/events/{run_date}/*.json",
        "file_format": "JSON",
        "write_disposition": "WRITE_TRUNCATE"
    }'
```

---

## Part 10 — BigQuery Data Transfer — Cross-Region & Dataset Copy

### Dataset Copy and Cross-Region Transfer

```bash
# ═══════════════════════════════════════════════════════════════
# CROSS-REGION DATASET COPY
# ═══════════════════════════════════════════════════════════════

# Copy dataset from US to EU (one-time)
bq mk --transfer_config \
    --data_source=cross_region_copy \
    --display_name="US to EU Copy" \
    --target_dataset=analytics_eu \
    --schedule="" \
    --params='{
        "source_dataset_id": "analytics",
        "source_project_id": "my-project",
        "overwrite_destination_table": "true"
    }'

# Recurring dataset copy (daily sync)
bq mk --transfer_config \
    --data_source=cross_region_copy \
    --display_name="Daily US→EU Sync" \
    --target_dataset=analytics_eu \
    --schedule="every 24 hours" \
    --params='{
        "source_dataset_id": "analytics",
        "source_project_id": "my-project",
        "overwrite_destination_table": "true"
    }'

# ═══════════════════════════════════════════════════════════════
# MANAGE TRANSFERS
# ═══════════════════════════════════════════════════════════════
# List transfers
bq ls --transfer_config --transfer_location=us

# Describe transfer
bq show --transfer_config TRANSFER_CONFIG_ID

# List transfer runs
bq ls --transfer_run --transfer_location=us TRANSFER_CONFIG_ID

# Trigger manual run
bq mk --transfer_run \
    --start_time=2024-01-15T00:00:00Z \
    --end_time=2024-01-16T00:00:00Z \
    TRANSFER_CONFIG_ID

# Delete transfer
bq rm --transfer_config TRANSFER_CONFIG_ID
```

---

## Part 11 — BigQuery Data Loading (Non-Transfer)

### Direct Loading Methods

```bash
# ═══════════════════════════════════════════════════════════════
# LOAD FROM GCS (one-time)
# ═══════════════════════════════════════════════════════════════

# CSV
bq load --source_format=CSV \
    --skip_leading_rows=1 \
    --autodetect \
    my_dataset.my_table \
    gs://my-bucket/data/*.csv

# JSON (newline-delimited)
bq load --source_format=NEWLINE_DELIMITED_JSON \
    --autodetect \
    my_dataset.my_table \
    gs://my-bucket/data/*.json

# Parquet
bq load --source_format=PARQUET \
    my_dataset.my_table \
    gs://my-bucket/data/*.parquet

# Avro
bq load --source_format=AVRO \
    my_dataset.my_table \
    gs://my-bucket/data/*.avro

# ORC
bq load --source_format=ORC \
    my_dataset.my_table \
    gs://my-bucket/data/*.orc

# With explicit schema
bq load --source_format=CSV \
    --skip_leading_rows=1 \
    my_dataset.my_table \
    gs://my-bucket/data.csv \
    name:STRING,age:INTEGER,email:STRING

# ═══════════════════════════════════════════════════════════════
# STREAMING INSERT (real-time)
# ═══════════════════════════════════════════════════════════════
# Python SDK
from google.cloud import bigquery
client = bigquery.Client()
errors = client.insert_rows_json(
    'my_dataset.my_table',
    [{'name': 'Alice', 'age': 30}, {'name': 'Bob', 'age': 25}],
)

# ═══════════════════════════════════════════════════════════════
# STORAGE WRITE API (high-throughput)
# ═══════════════════════════════════════════════════════════════
# Recommended over streaming inserts for high-volume
# Supports exactly-once semantics
# Lower cost than streaming inserts
```

### Loading Method Comparison

| Method | Use Case | Latency | Cost |
|--------|---------|---------|------|
| Batch load (bq load) | One-time, periodic loads | Seconds–minutes | Free |
| Streaming inserts | Real-time, low volume | Sub-second | $0.05/GB |
| Storage Write API | Real-time, high volume | Sub-second | $0.025/GB |
| BQDTS (scheduled) | Recurring automated loads | Minutes | Free |
| Federated query | Query in-place (no load) | Seconds | Query cost |

---

## Part 12 — Transfer Appliance (Physical)

### Offline Bulk Data Transfer

```
┌────────────────────────────────────────────────────────────────────┐
│         TRANSFER APPLIANCE WORKFLOW                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1: ORDER                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Console → Transfer Appliance → Create Order             │     │
│  │  Select: TA40 (40 TB) or TA300 (300 TB)                 │     │
│  │  Provide: shipping address, GCS destination bucket      │     │
│  │  Google ships racked appliance to your location          │     │
│  └──────────────────────────────────────────────────────────┘     │
│       │                                                             │
│       ▼                                                             │
│  Step 2: CAPTURE                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Connect appliance to your network (10/25/40 GbE)     │     │
│  │  • Mount NFS share on your servers                       │     │
│  │  • Copy data to appliance                                │     │
│  │  • Data encrypted with AES-256 (your key)               │     │
│  │  • Typical speed: ~40 Gbps sustained (TA300)            │     │
│  └──────────────────────────────────────────────────────────┘     │
│       │                                                             │
│       ▼                                                             │
│  Step 3: SHIP                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Finalize capture session                              │     │
│  │  • Ship appliance back to Google                        │     │
│  │  • Tracked shipping with tamper-evident packaging       │     │
│  └──────────────────────────────────────────────────────────┘     │
│       │                                                             │
│       ▼                                                             │
│  Step 4: UPLOAD                                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Google receives and connects appliance                │     │
│  │  • Data uploaded to your GCS bucket                     │     │
│  │  • You verify data integrity                            │     │
│  │  • Appliance securely wiped                             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Timeline:                                                          │
│  Order → Receive (3-5 days) → Capture (varies) → Ship (2-3 days) │
│  → Upload (1-2 days) → Verification                               │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Choosing the Right Transfer Method

### Decision Matrix

```
┌────────────────────────────────────────────────────────────────────┐
│         TRANSFER METHOD COMPARISON                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Method              │ Volume     │ Speed    │ Use When  │     │
│  ├──────────────────────┼────────────┼──────────┼───────────┤     │
│  │ gcloud storage cp    │ < 1 TB     │ Fast     │ Ad-hoc    │     │
│  │                      │            │          │ one-time  │     │
│  ├──────────────────────┼────────────┼──────────┼───────────┤     │
│  │ Storage Transfer Svc │ 1–100+ TB  │ Managed  │ Scheduled,│     │
│  │ (cloud-to-cloud)     │            │          │ recurring │     │
│  ├──────────────────────┼────────────┼──────────┼───────────┤     │
│  │ Storage Transfer Svc │ 1–100+ TB  │ Managed  │ On-prem   │     │
│  │ (agent, on-prem)     │            │          │ → GCS     │     │
│  ├──────────────────────┼────────────┼──────────┼───────────┤     │
│  │ Transfer Appliance   │ 20+ TB     │ Physical │ Slow net, │     │
│  │                      │            │ ship     │ bulk data │     │
│  ├──────────────────────┼────────────┼──────────┼───────────┤     │
│  │ BigQuery DTS         │ Any        │ Managed  │ SaaS → BQ │     │
│  │                      │            │          │ scheduled │     │
│  ├──────────────────────┼────────────┼──────────┼───────────┤     │
│  │ bq load              │ < 10 TB    │ Fast     │ One-time  │     │
│  │                      │            │          │ BQ load   │     │
│  ├──────────────────────┼────────────┼──────────┼───────────┤     │
│  │ Streaming / Write API│ Real-time  │ Real-time│ Continuous│     │
│  │                      │            │          │ ingestion │     │
│  └──────────────────────┴────────────┴──────────┴───────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Storage Transfer Job (S3 → GCS) ─────────────────────────
resource "google_storage_transfer_job" "s3_to_gcs" {
  description = "Daily S3 to GCS transfer"
  project     = var.project_id

  transfer_spec {
    aws_s3_data_source {
      bucket_name = "source-bucket"
      aws_access_key {
        access_key_id     = var.aws_access_key
        secret_access_key = var.aws_secret_key
      }
    }

    gcs_data_sink {
      bucket_name = google_storage_bucket.destination.name
      path        = "imported/"
    }

    object_conditions {
      include_prefixes          = ["data/"]
      exclude_prefixes          = ["data/temp/"]
      min_time_elapsed_since_last_modification = "3600s"
    }

    transfer_options {
      overwrite_objects_already_existing_in_sink = false
      delete_objects_from_source_after_transfer  = false
    }
  }

  schedule {
    schedule_start_date {
      year  = 2024
      month = 1
      day   = 15
    }
    start_time_of_day {
      hours   = 2
      minutes = 0
    }
    repeat_interval = "86400s"   # daily
  }
}

# ─── Storage Transfer Job (GCS → GCS) ────────────────────────
resource "google_storage_transfer_job" "gcs_to_gcs" {
  description = "Cross-region bucket sync"
  project     = var.project_id

  transfer_spec {
    gcs_data_source {
      bucket_name = "source-bucket-us"
      path        = "data/"
    }

    gcs_data_sink {
      bucket_name = "destination-bucket-eu"
      path        = "data/"
    }
  }

  schedule {
    schedule_start_date {
      year  = 2024
      month = 1
      day   = 1
    }
    repeat_interval = "3600s"   # hourly
  }
}

# ─── BigQuery Data Transfer Config ────────────────────────────
resource "google_bigquery_data_transfer_config" "gcs_load" {
  display_name           = "GCS Daily Load"
  data_source_id         = "google_cloud_storage"
  destination_dataset_id = google_bigquery_dataset.analytics.dataset_id
  location               = var.region
  schedule               = "every 24 hours"

  params = {
    destination_table_name_template = "events_{run_date}"
    data_path_template              = "gs://my-bucket/events/{run_date}/*.json"
    file_format                     = "JSON"
    write_disposition               = "WRITE_TRUNCATE"
  }
}

resource "google_bigquery_data_transfer_config" "cross_region" {
  display_name           = "US to EU Dataset Copy"
  data_source_id         = "cross_region_copy"
  destination_dataset_id = google_bigquery_dataset.analytics_eu.dataset_id
  location               = "eu"
  schedule               = "every 24 hours"

  params = {
    source_dataset_id              = "analytics"
    source_project_id              = var.project_id
    overwrite_destination_table    = "true"
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# STORAGE TRANSFER SERVICE
# ═══════════════════════════════════════════════════════════════
gcloud transfer jobs create SOURCE DEST [--opts]
gcloud transfer jobs list
gcloud transfer jobs describe JOB_NAME
gcloud transfer jobs run JOB_NAME
gcloud transfer jobs update JOB_NAME --schedule=""     # disable schedule
gcloud transfer jobs delete JOB_NAME
gcloud transfer operations list --job-names=JOB
gcloud transfer operations pause OP_NAME
gcloud transfer operations resume OP_NAME

# Agent management
gcloud transfer agent-pools create POOL_NAME
gcloud transfer agent-pools list
gcloud transfer agent-pools delete POOL_NAME

# ═══════════════════════════════════════════════════════════════
# GCLOUD STORAGE (file copy)
# ═══════════════════════════════════════════════════════════════
gcloud storage cp FILE gs://BUCKET/           # upload
gcloud storage cp gs://BUCKET/FILE ./         # download
gcloud storage cp -r DIR/ gs://BUCKET/        # recursive
gcloud storage rsync -r SOURCE DEST           # sync
gcloud storage mv gs://SRC/FILE gs://DEST/    # move

# ═══════════════════════════════════════════════════════════════
# BIGQUERY DATA TRANSFER
# ═══════════════════════════════════════════════════════════════
bq mk --transfer_config --data_source=SRC \
    --display_name=NAME --target_dataset=DS \
    --schedule=SCHEDULE --params='JSON'
bq ls --transfer_config --transfer_location=LOC
bq show --transfer_config CONFIG_ID
bq rm --transfer_config CONFIG_ID

# ═══════════════════════════════════════════════════════════════
# BIGQUERY LOAD
# ═══════════════════════════════════════════════════════════════
bq load --source_format=FMT [--autodetect] \
    DATASET.TABLE gs://BUCKET/FILE [SCHEMA]
bq extract DATASET.TABLE gs://BUCKET/export.csv
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Multi-Cloud Data Lake Consolidation

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: CONSOLIDATE DATA FROM AWS + AZURE + ON-PREM          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │                                                            │        │
│  │  AWS S3 (logs, events)                                    │        │
│  │  ──── Storage Transfer Service (daily) ────►              │        │
│  │                                              │             │        │
│  │  Azure Blob (documents, images)              │             │        │
│  │  ──── Storage Transfer Service (daily) ──────┤             │        │
│  │                                              │             │        │
│  │  On-Premises NAS (legacy data)               │             │        │
│  │  ──── Transfer Agent (nightly) ──────────────┤             │        │
│  │                                              ▼             │        │
│  │                                     ┌──────────────┐      │        │
│  │                                     │ GCS Bucket   │      │        │
│  │                                     │ (data lake)  │      │        │
│  │                                     └──────┬───────┘      │        │
│  │                                            │               │        │
│  │                                     ┌──────▼───────┐      │        │
│  │                                     │ BigQuery     │      │        │
│  │                                     │ (analytics)  │      │        │
│  │                                     └──────────────┘      │        │
│  │                                                            │        │
│  │  GCS lifecycle:                                           │        │
│  │  • Raw data (Standard) → 30 days                         │        │
│  │  • Processed (Nearline) → 90 days                        │        │
│  │  • Archive (Coldline) → 1 year                           │        │
│  │  • Delete after 3 years                                  │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Marketing Analytics Data Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: MARKETING DATA CONSOLIDATION                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  BigQuery Data Transfer Service (scheduled daily):        │        │
│  │                                                            │        │
│  │  Google Ads ──────────────────► ads_data dataset          │        │
│  │  YouTube Analytics ───────────► youtube_data dataset      │        │
│  │  Search Ads 360 ──────────────► search_data dataset       │        │
│  │  Campaign Manager ────────────► cm_data dataset           │        │
│  │                                                            │        │
│  │  Third-party (via STS → GCS → BQ):                       │        │
│  │  Facebook Ads API → Cloud Function → GCS → BQ load      │        │
│  │  LinkedIn Ads API → Cloud Function → GCS → BQ load      │        │
│  │                                                            │        │
│  │  Web analytics:                                           │        │
│  │  GA4 → BigQuery export (native integration, daily)       │        │
│  │                                                            │        │
│  │                      ┌──────────────────┐                 │        │
│  │  All sources ──────► │ BigQuery         │                 │        │
│  │                      │ unified_marketing│                 │        │
│  │                      │ dataset          │                 │        │
│  │                      └────────┬─────────┘                 │        │
│  │                               │                            │        │
│  │                      ┌────────▼─────────┐                 │        │
│  │                      │ Looker Studio    │                 │        │
│  │                      │ dashboards       │                 │        │
│  │                      └──────────────────┘                 │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Disaster Recovery Data Replication

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CROSS-REGION DR REPLICATION                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Primary (us-central1)           DR (europe-west1)        │        │
│  │                                                            │        │
│  │  GCS: app-data-us               GCS: app-data-eu         │        │
│  │  ──── STS (hourly sync) ────────────────────►            │        │
│  │                                                            │        │
│  │  BigQuery: analytics_us         BigQuery: analytics_eu   │        │
│  │  ──── BQDTS cross-region copy (daily) ──────►            │        │
│  │                                                            │        │
│  │  Cloud SQL: prod-db-us          Cloud SQL: prod-db-eu    │        │
│  │  ──── Cross-region read replica ────────────►            │        │
│  │                                                            │        │
│  │  On failover:                                             │        │
│  │  1. Promote Cloud SQL read replica                       │        │
│  │  2. Switch DNS to europe-west1 endpoints                 │        │
│  │  3. Applications use eu GCS & BQ datasets                │        │
│  │                                                            │        │
│  │  RPO: 1 hour (GCS), 24 hours (BigQuery), ~seconds (SQL) │        │
│  │  RTO: < 30 minutes (automated failover)                  │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Copy file to GCS | `gcloud storage cp FILE gs://BUCKET/` |
| Sync directory | `gcloud storage rsync -r DIR/ gs://BUCKET/` |
| Create STS job | `gcloud transfer jobs create SOURCE DEST` |
| Run STS job | `gcloud transfer jobs run JOB_NAME` |
| Install agent | `docker run ... gcr.io/cloud-ingest/tsop-agent` |
| BQ transfer | `bq mk --transfer_config --data_source=SRC ...` |
| BQ load file | `bq load --source_format=FMT DS.TBL gs://...` |
| List transfers | `bq ls --transfer_config --transfer_location=LOC` |

---

## When to Use Which Transfer Method? (Beginner Guide)

If you're new to GCP data transfers, use this simplified guide to pick the right tool.

### Decision Flowchart

```
┌────────────────────────────────────────────────────────────────────┐
│         WHICH TRANSFER METHOD SHOULD I USE?                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  How much data are you transferring?                               │
│       │                                                             │
│       ├── Less than 1 TB                                          │
│       │    └── ✅ gsutil / gcloud storage cp                      │
│       │        Simple CLI commands. Run from your machine          │
│       │        or a Compute Engine VM for faster speeds.           │
│       │                                                             │
│       ├── 1 TB – 10 TB                                            │
│       │    └── ✅ Storage Transfer Service                        │
│       │        Managed, resumable, scheduled. Handles retries     │
│       │        and parallelism automatically.                     │
│       │                                                             │
│       ├── 10+ TB (or slow/limited network)                        │
│       │    └── ✅ Transfer Appliance                              │
│       │        Google ships you a physical device. Load data      │
│       │        locally, ship it back. Best for massive datasets.  │
│       │                                                             │
│       └── Coming from another cloud (AWS S3 / Azure Blob)?       │
│            └── ✅ Storage Transfer Service                        │
│                Built-in connectors for S3 and Azure Blob.         │
│                No agents needed — just provide credentials.       │
│                                                                      │
│  Transferring structured data into BigQuery?                       │
│       │                                                             │
│       ├── From SaaS sources (Google Ads, YouTube, etc.)           │
│       │    └── ✅ BigQuery Data Transfer Service                  │
│       │                                                             │
│       └── From GCS / S3 / other datasets                          │
│            └── ✅ BigQuery Data Transfer Service                  │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Quick Recommendation Table

| Scenario | Recommended Tool | Why? |
|----------|-----------------|------|
| Copy a few files (< 1 TB) | `gcloud storage cp` / `gsutil` | Simple, fast, no setup needed |
| Regular sync from on-prem (1–10 TB) | Storage Transfer Service (with agent) | Managed, resumable, scheduled |
| One-time large transfer (10+ TB) | Transfer Appliance | Physical shipping avoids network limits |
| AWS S3 → GCS | Storage Transfer Service | Native S3 connector, no agents |
| Azure Blob → GCS | Storage Transfer Service | Native Azure connector |
| GCS bucket → GCS bucket | Storage Transfer Service | Cross-region/project copies |
| Google Ads data → BigQuery | BigQuery Data Transfer Service | Pre-built SaaS connector |
| CSV/JSON in GCS → BigQuery | `bq load` or BigQuery console | Simple one-time loads |
| Real-time streaming → BigQuery | Pub/Sub + Dataflow | Not a "transfer" — use streaming pipeline |
| Database → Cloud SQL | Database Migration Service (Ch 60) | Purpose-built for DB migration |

> **Rule of thumb:** If it fits through the network in a reasonable time, use Storage Transfer Service. If not, use Transfer Appliance. For quick ad-hoc copies, CLI tools (`gcloud storage cp`) are the simplest option.

---

## Console Walkthrough: Creating Transfer Jobs

### Creating a Storage Transfer Job

```
┌────────────────────────────────────────────────────────────────────┐
│  Console → Data Transfer → Storage Transfer Service → CREATE       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Go to Console → Data Transfer → Storage Transfer Service      │
│     (Navigation menu → Data Transfer → Transfer jobs)             │
│                                                                      │
│  2. Click "+ CREATE TRANSFER JOB"                                 │
│                                                                      │
│  3. Source:                                                         │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  Source type:  Google Cloud Storage                       │  │
│     │               Amazon S3                                   │  │
│     │               Azure Blob Storage                          │  │
│     │               POSIX filesystem (on-prem with agent)       │  │
│     │                                                           │  │
│     │  Bucket/container name: source-bucket-name                │  │
│     │  (For S3/Azure: provide access key & secret)              │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  4. Destination:                                                    │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  Destination bucket: my-gcs-destination-bucket            │  │
│     │  (Optional) Path prefix: data/imported/                   │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  5. Schedule:                                                       │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  ○ Run once (starting now or at a specific time)         │  │
│     │  ○ Run every day / week / custom schedule                │  │
│     │  ○ Run on demand (trigger manually later)                │  │
│     │                                                           │  │
│     │  Start date: 2026-05-22                                   │  │
│     │  Start time: 02:00 AM (off-peak recommended)             │  │
│     │  End date:   (optional) leave blank for ongoing          │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  6. Transfer options:                                               │
│     • Overwrite: if different / always / never                    │
│     • Delete from source: after transfer (optional)               │
│     • Delete from destination: if not in source (optional)        │
│     • Include/exclude prefixes: filter specific paths             │
│                                                                      │
│  7. Click "CREATE" → job starts based on schedule                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Creating a BigQuery Data Transfer

```
┌────────────────────────────────────────────────────────────────────┐
│  Console → BigQuery → Data Transfers → CREATE                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Go to Console → BigQuery → Data Transfers                     │
│     (Navigation menu → BigQuery → Data transfers)                 │
│                                                                      │
│  2. Click "+ CREATE TRANSFER"                                     │
│                                                                      │
│  3. Source type:                                                    │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  Choose from:                                             │  │
│     │  • Google Ads                                             │  │
│     │  • Google Ad Manager                                      │  │
│     │  • YouTube Channel / Content Owner                        │  │
│     │  • Amazon S3                                              │  │
│     │  • Google Cloud Storage                                   │  │
│     │  • Cross-region dataset copy                              │  │
│     │  • Scheduled query                                        │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  4. Transfer config:                                                │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  Transfer name:  daily-ads-import                         │  │
│     │  Destination dataset: my_analytics_dataset                │  │
│     │  Schedule: every 24 hours (or custom cron)               │  │
│     │                                                           │  │
│     │  Source-specific settings:                                │  │
│     │  (e.g., for S3: bucket URI, access key, file format)     │  │
│     │  (e.g., for Ads: customer ID, link to Ads account)       │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  5. Notification preferences (optional):                           │
│     • Email on failure                                             │
│     • Pub/Sub topic for run notifications                         │
│                                                                      │
│  6. Click "SAVE" → transfer runs on schedule                      │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Monitoring Transfer Progress

```
┌────────────────────────────────────────────────────────────────────┐
│  Console → Monitor Your Transfers                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Storage Transfer Service:                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Console → Data Transfer → Transfer jobs → [Your Job]    │     │
│  │                                                           │     │
│  │  Status indicators:                                       │     │
│  │  ● IN PROGRESS — transfer is actively running            │     │
│  │  ● SUCCESS     — all objects transferred ✓               │     │
│  │  ● FAILED      — check error log for details             │     │
│  │  ● ABORTED     — manually stopped or timed out           │     │
│  │                                                           │     │
│  │  Details tab shows:                                       │     │
│  │  • Objects found / transferred / skipped / failed        │     │
│  │  • Bytes transferred                                     │     │
│  │  • Start time and duration                               │     │
│  │  • Error log (if any failures)                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  BigQuery Data Transfer:                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Console → BigQuery → Data transfers → [Your Transfer]   │     │
│  │                                                           │     │
│  │  Run history shows:                                       │     │
│  │  • Each scheduled run with status (SUCCEEDED / FAILED)   │     │
│  │  • Run time and duration                                 │     │
│  │  • Click a run → view detailed logs                      │     │
│  │  • Destination table(s) created/updated                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Deleting a Transfer Job

```
┌────────────────────────────────────────────────────────────────────┐
│  Console → Delete a Transfer Job                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Storage Transfer Service:                                         │
│  1. Console → Data Transfer → Transfer jobs                       │
│  2. Click the job you want to delete                               │
│  3. Click "DELETE" (trash icon at the top)                        │
│  4. Confirm deletion                                               │
│  Note: Already-transferred data in the destination is NOT deleted │
│                                                                      │
│  BigQuery Data Transfer:                                           │
│  1. Console → BigQuery → Data transfers                           │
│  2. Click the transfer configuration                               │
│  3. Click "DELETE" → confirm                                      │
│  Note: Already-loaded tables in BigQuery are NOT deleted          │
│                                                                      │
│  CLI alternative:                                                   │
│  gcloud transfer jobs delete JOB_NAME                             │
│  bq rm --transfer_config TRANSFER_CONFIG_ID                       │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

> **Important:** Deleting a transfer job stops future scheduled runs but does NOT delete any data that was already transferred. Your destination bucket or BigQuery dataset remains unchanged.

---

## What's Next?

Continue to **Chapter 62: Google Cloud Architecture Framework** → `62-architecture-framework.md`
