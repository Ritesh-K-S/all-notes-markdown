# Chapter 52: Amazon Kinesis

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Kinesis Fundamentals](#part-1-kinesis-fundamentals)
- [Part 2: Kinesis Data Streams (Full Portal Walkthrough)](#part-2-kinesis-data-streams-full-portal-walkthrough)
- [Part 3: Kinesis Data Firehose](#part-3-kinesis-data-firehose)
- [Part 4: Kinesis Data Analytics](#part-4-kinesis-data-analytics)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Real-Time Streaming? Why Do We Need Kinesis?

Imagine two ways to watch a sports game:
- 📰 **Batch processing**: Read the results in tomorrow's newspaper (data is hours/days old)
- 📺 **Real-time streaming**: Watch the live broadcast (data arrives as it happens)

**Kinesis is for live broadcast scenarios** — when you need to process data **as it arrives**, not hours later.

**Simple real-world examples:**
- 📱 A ride-sharing app processes millions of GPS location updates per second to match drivers with riders
- 🎮 A gaming platform analyzes player actions in real-time to detect cheating
- 📊 A trading platform processes stock price changes within milliseconds
- 🌐 A website tracks user clicks in real-time to update a live dashboard

**When to use Kinesis vs SQS:**
- **SQS**: Process one message at a time, message is deleted after processing (task queue)
- **Kinesis**: Process a continuous stream of data, multiple consumers can read the same data, data retained for replay

Amazon Kinesis is a family of services for collecting, processing, and analyzing real-time streaming data. It enables you to process data as it arrives rather than waiting for batch processing.

```
What you'll learn:
├── Kinesis family overview (Streams, Firehose, Analytics)
├── Kinesis Data Streams (portal walkthrough)
├── Kinesis Data Firehose (delivery stream walkthrough)
├── Kinesis Data Analytics (SQL & Flink)
├── Producers & consumers
└── Terraform & CLI examples
```

---

## Part 1: Kinesis Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           KINESIS FAMILY                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│ │ Kinesis Data     │  │ Kinesis Data     │  │ Kinesis Data     │ │
│ │ Streams          │  │ Firehose         │  │ Analytics        │ │
│ ├──────────────────┤  ├──────────────────┤  ├──────────────────┤ │
│ │ Real-time ingest │  │ Deliver to       │  │ Process with     │ │
│ │ & process with   │  │ destinations     │  │ SQL or Apache    │ │
│ │ custom consumers │  │ (S3, Redshift,   │  │ Flink            │ │
│ │                  │  │  OpenSearch, etc.)│  │                  │ │
│ │ You manage:      │  │ Fully managed:   │  │ Fully managed:   │ │
│ │ Shards, capacity │  │ Auto-scaling,    │  │ Process streams  │ │
│ │                  │  │ no management    │  │ in real-time     │ │
│ │ Latency: ~200ms  │  │ Latency: 60s+   │  │                  │ │
│ │                  │  │ (near real-time) │  │                  │ │
│ └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│                                                                       │
│ Data flow:                                                           │
│ Producers → Kinesis Data Streams → Consumers                      │
│                    └─→ Kinesis Data Firehose → S3/Redshift        │
│                    └─→ Kinesis Data Analytics → Real-time SQL     │
│                                                                       │
│ Kinesis vs SQS:                                                      │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ Kinesis       │ SQS                  │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Model                │ Stream (log)  │ Queue                │   │
│ │ Consumers            │ Multiple      │ Single (per message) │   │
│ │ Retention            │ 1-365 days    │ 1 min-14 days        │   │
│ │ Ordering             │ Per shard     │ Best-effort/FIFO     │   │
│ │ Replay               │ ✅ (re-read)  │ ❌ (consumed once)   │   │
│ │ Throughput           │ 1 MB/s/shard  │ Unlimited            │   │
│ │ Use case             │ Analytics,    │ Task queue,          │   │
│ │                      │ IoT, logs     │ decoupling           │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Kinesis Data Streams (Full Portal Walkthrough)

```
Console → Kinesis → Data streams → Create data stream

┌─────────────────────────────────────────────────────────────────┐
│           CREATE DATA STREAM                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Data stream name: [clickstream-events]                        │
│                                                                   │
│ Capacity mode:                                                 │
│ ● On-demand (⚡ recommended to start)                        │
│   → Automatically scales up/down                             │
│   → Up to 200 MB/s write, 400 MB/s read                    │
│   → Pay per GB ingested + retrieved                         │
│   → ⚡ No capacity planning needed                           │
│                                                                   │
│ ○ Provisioned                                                 │
│   → You manage number of shards                              │
│   → Each shard: 1 MB/s in, 2 MB/s out                      │
│   → 1000 records/s per shard (write)                        │
│   → Pay per shard-hour ($0.015/hour)                        │
│   → ⚡ Predictable cost for steady workloads                │
│                                                                   │
│   Number of shards: [4]                                      │
│   → Write capacity: 4 MB/s, 4000 records/s                 │
│   → Read capacity: 8 MB/s                                   │
│                                                                   │
│ Data retention:                                                │
│ [24] hours (range: 24 hours to 365 days)                     │
│ → Default 24 hours (free)                                    │
│ → Extended retention: Additional cost                       │
│ → ⚡ Longer retention = replay older data                    │
│                                                                   │
│ Encryption:                                                    │
│ ● Server-side encryption: ☑ Enabled                         │
│ KMS key: [aws/kinesis ▼] (or customer managed)              │
│                                                                   │
│ Tags:                                                          │
│ [Environment] = [Production]                                  │
│                                                                   │
│                    [Create data stream]                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STREAM CONCEPTS                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Shard model:                                                   │
│ ┌──────────────────────────────────────────┐                  │
│ │ Stream: clickstream-events               │                  │
│ │ ┌──────────┐                             │                  │
│ │ │ Shard 1  │← Partition key hash range  │                  │
│ │ │ 1 MB/s   │  Records: A, D, G          │                  │
│ │ └──────────┘                             │                  │
│ │ ┌──────────┐                             │                  │
│ │ │ Shard 2  │← Partition key hash range  │                  │
│ │ │ 1 MB/s   │  Records: B, E, H          │                  │
│ │ └──────────┘                             │                  │
│ │ ┌──────────┐                             │                  │
│ │ │ Shard 3  │← Partition key hash range  │                  │
│ │ │ 1 MB/s   │  Records: C, F, I          │                  │
│ │ └──────────┘                             │                  │
│ └──────────────────────────────────────────┘                  │
│                                                                   │
│ Record:                                                        │
│ ├── Partition key: Determines which shard (hash-based)      │
│ │   → Same key = same shard = ordered                       │
│ │   → e.g., user-id as partition key                        │
│ ├── Data blob: Up to 1 MB                                   │
│ └── Sequence number: Unique per record (assigned by Kinesis)│
│                                                                   │
│ Producers (send data):                                        │
│ ├── AWS SDK (PutRecord / PutRecords)                        │
│ ├── Kinesis Producer Library (KPL) - batching + aggregation │
│ ├── Kinesis Agent (log file streaming)                      │
│ ├── CloudWatch Logs subscription                            │
│ └── IoT Core rules                                           │
│                                                                   │
│ Consumers (read data):                                        │
│ ├── Kinesis Client Library (KCL) - managed checkpointing   │
│ ├── AWS Lambda (event source mapping)                       │
│ ├── Kinesis Data Firehose                                   │
│ ├── Kinesis Data Analytics                                  │
│ └── AWS SDK (GetRecords)                                    │
│                                                                   │
│ Enhanced Fan-Out:                                              │
│ ├── Dedicated 2 MB/s throughput per consumer                │
│ ├── Push-based (HTTP/2, ~70ms latency)                     │
│ ├── Up to 20 enhanced consumers per stream                  │
│ └── ⚡ Use for multiple consumers reading same stream       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Kinesis Data Firehose

```
Console → Kinesis → Delivery streams → Create delivery stream

┌─────────────────────────────────────────────────────────────────┐
│           CREATE DELIVERY STREAM                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Source:                                                         │
│ ● Direct PUT (⚡ apps send directly to Firehose)             │
│ ○ Kinesis Data Stream (read from existing stream)            │
│ ○ Amazon MSK (read from Kafka topic)                         │
│                                                                   │
│ Delivery stream name: [clickstream-to-s3]                    │
│                                                                   │
│ Destination:                                                   │
│ ● Amazon S3                                                    │
│ ○ Amazon Redshift (via S3 + COPY)                            │
│ ○ Amazon OpenSearch Service                                   │
│ ○ Splunk                                                      │
│ ○ HTTP endpoint (custom)                                     │
│ ○ MongoDB Cloud                                               │
│ ○ Datadog / New Relic / etc.                                 │
│                                                                   │
│ Transform and convert records (optional):                    │
│ ☑ Enable data transformation                                 │
│ Lambda function: [transform-clickstream ▼]                   │
│ → Transform/filter/enrich records before delivery           │
│ Buffer size: [1] MB (for Lambda invocation)                  │
│                                                                   │
│ Record format conversion (optional):                          │
│ ☐ Convert to Apache Parquet or ORC                          │
│   → Schema from AWS Glue table                              │
│   → ⚡ Parquet for analytics (Athena, Redshift Spectrum)    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           S3 DESTINATION SETTINGS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ S3 bucket: [clickstream-data-bucket ▼]                       │
│                                                                   │
│ S3 prefix: [data/year=!{timestamp:yyyy}/month=!{timestamp:MM}/│
│ day=!{timestamp:dd}/hour=!{timestamp:HH}/]                   │
│ → ⚡ Partition by date for efficient Athena queries          │
│                                                                   │
│ S3 error prefix: [errors/year=!{timestamp:yyyy}/]            │
│                                                                   │
│ Buffer conditions:                                             │
│ Buffer size: [5] MB (1-128 MB)                               │
│ Buffer interval: [300] seconds (60-900 seconds)              │
│ → Firehose delivers whichever threshold is hit first       │
│ → Smaller = more files, lower latency                       │
│ → Larger = fewer files, higher latency                      │
│ → ⚡ 5 MB / 300s is a good default                          │
│                                                                   │
│ Compression: [GZIP ▼]                                        │
│ ├── GZIP (⚡ good balance)                                  │
│ ├── Snappy (faster, less compression)                       │
│ ├── ZIP                                                      │
│ └── Uncompressed                                             │
│                                                                   │
│ Encryption: ☑ Enabled (SSE-S3 or SSE-KMS)                  │
│                                                                   │
│ Backup: ☑ Enable source record backup to S3                 │
│ → Stores raw records before transformation                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Kinesis Data Analytics

```
┌─────────────────────────────────────────────────────────────────────┐
│           KINESIS DATA ANALYTICS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two flavors:                                                         │
│                                                                       │
│ 1. SQL Applications (legacy, being replaced by Managed Flink):    │
│ ├── Write SQL queries against streaming data                     │
│ ├── Tumbling/sliding windows, aggregations                      │
│ ├── Input: Kinesis Data Streams or Firehose                    │
│ └── Output: Kinesis Streams, Firehose, Lambda                  │
│                                                                       │
│ 2. Apache Managed Flink (⚡ recommended):                          │
│ ├── Apache Flink on AWS (managed)                                │
│ ├── Write in Java, Scala, Python, or SQL                        │
│ ├── Complex event processing (CEP)                               │
│ ├── Stateful stream processing                                   │
│ ├── Exactly-once processing semantics                           │
│ └── Integrates with: Kinesis, Kafka, S3                        │
│                                                                       │
│ Console → Kinesis → Analytics applications → Create                │
│                                                                       │
│ Runtime: [Apache Flink 1.18 ▼]                                     │
│ Application name: [clickstream-analytics]                          │
│                                                                       │
│ Application code:                                                    │
│ ● Upload from S3 (JAR file)                                       │
│ ○ Apache Zeppelin notebook (interactive development)              │
│                                                                       │
│ Scaling:                                                              │
│ Parallelism: [4]                                                    │
│ Parallelism per KPU: [1]                                            │
│ ☑ Enable auto-scaling                                               │
│                                                                       │
│ Example SQL (legacy SQL app):                                       │
│ CREATE OR REPLACE STREAM "CLICK_COUNTS" (                          │
│   page VARCHAR(100),                                                │
│   click_count INTEGER,                                              │
│   window_end TIMESTAMP                                              │
│ );                                                                    │
│ INSERT INTO "CLICK_COUNTS"                                          │
│ SELECT page,                                                         │
│        COUNT(*) AS click_count,                                     │
│        TUMBLE_END(event_time, INTERVAL '1' MINUTE)                │
│ FROM "SOURCE_STREAM"                                                │
│ GROUP BY page, TUMBLE(event_time, INTERVAL '1' MINUTE);          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# Kinesis Data Stream
resource "aws_kinesis_stream" "clickstream" {
  name             = "clickstream-events"
  shard_count      = 4  # For provisioned mode
  retention_period = 24 # hours

  stream_mode_details {
    stream_mode = "PROVISIONED"  # or ON_DEMAND
  }

  encryption_type = "KMS"
  kms_key_id      = "alias/aws/kinesis"

  shard_level_metrics = [
    "IncomingBytes",
    "OutgoingBytes",
    "IncomingRecords",
    "OutgoingRecords"
  ]
}

# Kinesis Firehose to S3
resource "aws_kinesis_firehose_delivery_stream" "s3" {
  name        = "clickstream-to-s3"
  destination = "extended_s3"

  kinesis_source_configuration {
    kinesis_stream_arn = aws_kinesis_stream.clickstream.arn
    role_arn           = aws_iam_role.firehose.arn
  }

  extended_s3_configuration {
    role_arn           = aws_iam_role.firehose.arn
    bucket_arn         = aws_s3_bucket.data.arn
    prefix             = "data/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/"
    error_output_prefix = "errors/"
    buffering_size     = 5    # MB
    buffering_interval = 300  # seconds
    compression_format = "GZIP"

    processing_configuration {
      enabled = true
      processors {
        type = "Lambda"
        parameters {
          parameter_name  = "LambdaArn"
          parameter_value = "${aws_lambda_function.transform.arn}:$LATEST"
        }
      }
    }
  }
}

# Lambda event source mapping for Kinesis
resource "aws_lambda_event_source_mapping" "kinesis" {
  event_source_arn  = aws_kinesis_stream.clickstream.arn
  function_name     = aws_lambda_function.processor.arn
  starting_position = "LATEST"
  batch_size        = 100
  parallelization_factor = 10
  maximum_batching_window_in_seconds = 5

  bisect_batch_on_function_error = true
  maximum_retry_attempts         = 3

  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.dlq.arn
    }
  }
}
```

```bash
# Create stream
aws kinesis create-stream --stream-name clickstream-events --shard-count 4

# Put record
aws kinesis put-record \
  --stream-name clickstream-events \
  --partition-key "user-123" \
  --data '{"event":"click","page":"/products","timestamp":"2024-01-15T10:30:00Z"}'

# Get shard iterator
aws kinesis get-shard-iterator \
  --stream-name clickstream-events \
  --shard-id shardId-000000000000 \
  --shard-iterator-type LATEST

# Get records
aws kinesis get-records --shard-iterator "AAA..."

# Describe stream
aws kinesis describe-stream-summary --stream-name clickstream-events

# Put to Firehose
aws firehose put-record \
  --delivery-stream-name clickstream-to-s3 \
  --record '{"Data":"eyJldmVudCI6ImNsaWNrIn0="}'
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Real-Time Analytics Pipeline                             │
│ Web App → Kinesis Data Streams → Lambda (process) → DynamoDB     │
│                                → Firehose → S3 → Athena           │
│ ├── Real-time: Lambda processes events instantly                │
│ ├── Batch: Firehose delivers to S3 for Athena queries          │
│ └── Multiple consumers on same stream                            │
│                                                                       │
│ Pattern 2: Log Aggregation                                          │
│ CloudWatch Logs → Subscription → Kinesis Firehose → S3          │
│                                                → OpenSearch       │
│ ├── Centralize logs from all services                            │
│ ├── Firehose handles batching and delivery                      │
│ ├── Convert to Parquet for cost-effective storage              │
│ └── Search in OpenSearch, query in Athena                       │
│                                                                       │
│ Pattern 3: IoT Data Ingestion                                       │
│ IoT Devices → IoT Core → Kinesis Data Streams → Analytics      │
│ ├── Millions of devices, high throughput                        │
│ ├── Partition by device-id (ordered per device)                │
│ ├── Real-time anomaly detection                                  │
│ └── Archive to S3 for ML training                                │
│                                                                       │
│ Pattern 4: Event Sourcing                                            │
│ Application → Kinesis (event log) → Multiple consumers          │
│ ├── Read models (CQRS pattern)                                   │
│ ├── Audit trail (long retention)                                │
│ ├── Replay events to rebuild state                              │
│ └── ⚡ 365-day retention for complete event history             │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use on-demand mode unless you have predictable throughput    │
│ 2. Choose partition keys carefully (avoid hot shards)           │
│ 3. Use Enhanced Fan-Out for multiple consumers                  │
│ 4. Firehose for simple delivery (S3, Redshift, OpenSearch)     │
│ 5. Lambda + Kinesis: Enable bisect on error + DLQ              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Kinesis Quick Reference:
├── Data Streams: Real-time ingestion (custom consumers)
│   ├── Provisioned: 1 MB/s write, 2 MB/s read per shard
│   ├── On-demand: Auto-scaling, up to 200 MB/s
│   ├── Retention: 24 hours to 365 days
│   └── Enhanced Fan-Out: 2 MB/s per consumer (dedicated)
├── Data Firehose: Near real-time delivery (managed)
│   ├── Destinations: S3, Redshift, OpenSearch, HTTP, Splunk
│   ├── Buffer: 1-128 MB or 60-900 seconds
│   ├── Transform: Lambda, convert to Parquet/ORC
│   └── No shard management needed
├── Data Analytics: Stream processing (Flink/SQL)
│   └── Real-time aggregation, windowing, CEP
├── Video Streams: Real-time video ingestion & processing
│   ├── Use cases: Security cameras, smart home, video analytics
│   ├── Consumers: Rekognition Video, SageMaker, custom ML
│   └── Retention: 1 hour to 10 years
├── Producers: SDK, KPL, Agent, IoT, CloudWatch Logs
├── Consumers: KCL, Lambda, Firehose, Analytics
├── ⚡ Kinesis for streaming analytics, SQS for task queues
├── ⚡ Firehose for simple S3/Redshift delivery
└── ⚡ Partition key determines ordering and shard placement
```

### Troubleshooting: Common Kinesis Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `ProvisionedThroughputExceededException` | Hot shard (one partition key gets all traffic) | Distribute partition keys evenly or use on-demand |
| Iterator age increasing | Consumers falling behind producers | Add more shards, use enhanced fan-out, or add consumers |
| Duplicate records | KPL retries, consumer retries | Make consumer processing idempotent |
| Data loss | Record too large (>1 MB) | Compress or split large records |
| Firehose data gaps | Lambda transform errors | Check CloudWatch for transform errors, fix Lambda |

---

## What's Next?

In **Chapter 53: Athena & Glue**, we'll cover serverless SQL queries on S3 data, ETL pipelines with Glue, the Data Catalog, and crawlers for schema discovery.
