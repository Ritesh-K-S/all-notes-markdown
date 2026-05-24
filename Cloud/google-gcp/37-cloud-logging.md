# Chapter 37 — Cloud Logging

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Fundamentals](#part-1--fundamentals)
- [Part 2: Architecture & Core Concepts](#part-2--architecture--core-concepts)
- [Part 3: Log Entry Structure](#part-3--log-entry-structure)
- [Part 4: Log Explorer](#part-4--log-explorer)
- [Part 5: Log Router & Sinks](#part-5--log-router--sinks)
- [Part 6: Exclusion Filters](#part-6--exclusion-filters)
- [Part 7: Log Buckets & Views](#part-7--log-buckets--views)
- [Part 8: Log-Based Metrics](#part-8--log-based-metrics)
- [Part 9: Log-Based Alerts](#part-9--log-based-alerts)
- [Part 10: Log Analytics](#part-10--log-analytics)
- [Part 11: Audit Logs](#part-11--audit-logs)
- [Part 12: Application & Access Logs](#part-12--application--access-logs)
- [Part 13: Ops Agent & Structured Logging](#part-13--ops-agent--structured-logging)
- [Part 14: Console Walkthrough — Log Sinks, Buckets & Metrics](#part-14-console-walkthrough--log-sinks-buckets--metrics)
- [Part 15: Terraform & gcloud CLI Reference](#part-15--terraform--gcloud-cli-reference)
- [Part 16: Real-World Patterns](#part-16--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Logging (formerly Stackdriver Logging) is a fully managed, real-time log-management service that allows you to store, search, analyze, monitor, and alert on log data from GCP services, applications, and on-premises environments. With the Log Router at its centre, Cloud Logging lets you route logs to different destinations—buckets, BigQuery, Pub/Sub, or Cloud Storage—while exclusion filters control cost by dropping unwanted entries before they are stored.

---

## Part 1 — Fundamentals

### What Is Cloud Logging?

Cloud Logging ingests log entries from more than 150 GCP services automatically and from any custom source via the API, agents, or client libraries. It provides a centralised platform to explore, analyse, alert on, and archive log data.

```
┌─────────────────────────────────────────────────────────────────┐
│                CLOUD LOGGING OVERVIEW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Sources                                                         │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │ GCP        │ │ Applications│ │ On-Prem /  │ │ Third-     │  │
│  │ Services   │ │ (custom)   │ │ Multi-Cloud│ │ Party      │  │
│  │ (auto)     │ │            │ │            │ │            │  │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘  │
│        └───────────────┼──────────────┼───────────────┘          │
│                        ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Cloud Logging API                       │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Log Router                            │    │
│  │  ┌──────────────┐  ┌──────────────┐                     │    │
│  │  │ Inclusion    │  │ Exclusion    │                     │    │
│  │  │ Filters      │  │ Filters      │                     │    │
│  │  │ (sinks)      │  │ (drop logs)  │                     │    │
│  │  └──────┬───────┘  └──────────────┘                     │    │
│  └─────────┼───────────────────────────────────────────────┘    │
│            │                                                     │
│    ┌───────┼──────────┬──────────────┬───────────────┐          │
│    ▼       ▼          ▼              ▼               ▼          │
│  ┌─────┐ ┌─────┐  ┌────────┐  ┌──────────┐  ┌───────────┐    │
│  │Log  │ │Log  │  │BigQuery│  │Cloud     │  │Pub/Sub    │    │
│  │Buck-│ │Buck-│  │Dataset │  │Storage   │  │Topic      │    │
│  │et   │ │et   │  │        │  │Bucket    │  │           │    │
│  │_Req │ │_Def │  │(analyt)│  │(archive) │  │(stream)   │    │
│  └─────┘ └─────┘  └────────┘  └──────────┘  └───────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Capabilities

| Capability | Description |
|------------|-------------|
| **Log Explorer** | Interactive UI for searching, filtering, and viewing logs |
| **Log Router** | Routes every log entry through configurable sinks and filters |
| **Sinks** | Send logs to buckets, BigQuery, Cloud Storage, or Pub/Sub |
| **Exclusion filters** | Drop unwanted logs before storage to reduce cost |
| **Log buckets** | Storage containers with configurable retention |
| **Log views** | Control who can see which logs within a bucket |
| **Log Analytics** | SQL-based querying of logs (BigQuery-linked buckets) |
| **Log-based metrics** | Turn log patterns into Cloud Monitoring metrics |
| **Log-based alerts** | Alert directly when a log entry matches a filter |
| **Audit Logs** | Who did what, where, and when across GCP |

### Cross-Cloud Comparison

| Feature | GCP Cloud Logging | AWS CloudWatch Logs | Azure Monitor Logs |
|---------|-------------------|--------------------|--------------------|
| Service | Cloud Logging | CloudWatch Logs | Log Analytics workspace |
| Query language | Logging query language | CloudWatch Logs Insights | KQL (Kusto) |
| SQL querying | Log Analytics (BigQuery SQL) | — | KQL |
| Router | Log Router (sinks + exclusions) | Subscription filters | Diagnostic settings |
| Archive | Sink → Cloud Storage | Export → S3 | Export → Storage Account |
| Analytics | Sink → BigQuery | Export → S3 → Athena | KQL in workspace |
| Streaming | Sink → Pub/Sub | Subscription → Kinesis/Lambda | Event Hub |
| Audit logs | Admin Activity, Data Access, System Event, Policy Denied | CloudTrail | Activity Log |
| Metrics from logs | Log-based metrics | Metric filters | Custom log queries → alerts |
| Agent | Ops Agent / Logging agent | CloudWatch Agent | Azure Monitor Agent |
| Structured logging | JSON payloads, special fields | JSON → fields | JSON → dynamic fields |
| Retention | 30 days default (configurable 1–3,650 days) | 1 day–10 years | 30 days–2 years |
| Free tier | 50 GB/month ingestion | 5 GB ingest + 5 GB storage | 5 GB/month ingest |

### Pricing Overview

| Component | Cost |
|-----------|------|
| Ingestion — first 50 GB/month | Free |
| Ingestion — beyond 50 GB | ~$0.50/GiB |
| Storage — `_Required` bucket (400-day retention) | Free |
| Storage — `_Default` bucket (30-day default) | Free within retention |
| Storage — custom bucket (beyond default retention) | ~$0.01/GiB/month |
| Log Analytics (BigQuery-linked) | Standard BigQuery pricing for queries |
| Log Router sinks (routing) | No additional charge |
| Exclusion filters | Excluded logs are not charged |
| Admin Activity audit logs | Always free (ingestion and 400-day storage) |
| Data Access audit logs | Charged at standard ingestion rate |

> **Cost control tip**: Use exclusion filters to drop high-volume, low-value logs (e.g., health-check access logs, debug-level entries in production) before they are stored.

---

## Part 2 — Architecture & Core Concepts

### The Log Router

Every log entry written to Cloud Logging passes through the **Log Router**. The router evaluates each entry against all configured sinks and exclusion filters:

```
┌──────────────────────────────────────────────────────────────┐
│                    LOG ROUTER FLOW                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Log Entry arrives                                             │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────────────────────────────────┐                    │
│  │  Log Router                          │                    │
│  │                                      │                    │
│  │  1. Check _Required sink             │                    │
│  │     (Admin Activity, System Event,   │                    │
│  │      Access Transparency)            │                    │
│  │     → Always stored, cannot exclude  │                    │
│  │                                      │                    │
│  │  2. Check exclusion filters          │                    │
│  │     → Matching entries are dropped   │                    │
│  │                                      │                    │
│  │  3. Check _Default sink              │                    │
│  │     → Stores in _Default bucket      │                    │
│  │     (can be disabled or filtered)    │                    │
│  │                                      │                    │
│  │  4. Check user-defined sinks         │                    │
│  │     → Route to destinations          │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
│  Sink destinations:                                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Log      │ │ BigQuery │ │ Cloud    │ │ Pub/Sub  │       │
│  │ Bucket   │ │ Dataset  │ │ Storage  │ │ Topic    │       │
│  │(same/    │ │          │ │ Bucket   │ │          │       │
│  │ other    │ │          │ │          │ │          │       │
│  │ project) │ │          │ │          │ │          │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                                │
│  Note: A single log entry can match MULTIPLE sinks           │
│  (entries are copied, not moved)                              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Built-In Sinks and Buckets

| Sink / Bucket | Description | Retention | Modifiable? |
|---------------|-------------|-----------|-------------|
| `_Required` sink → `_Required` bucket | Admin Activity, System Event, Access Transparency logs | 400 days | Cannot disable, exclude, or change retention |
| `_Default` sink → `_Default` bucket | All other logs (unless excluded) | 30 days (configurable 1–3,650) | Can disable sink, add exclusions, change retention |

### Sink Scope Levels

```
┌──────────────────────────────────────────────────────────────┐
│                  SINK SCOPE LEVELS                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Organization-level sink                                       │
│  ├── Routes logs from ALL projects in the org                │
│  ├── Requires roles/logging.configWriter at org level        │
│  └── Aggregated sinks for compliance/security                │
│                                                                │
│  Folder-level sink                                             │
│  ├── Routes logs from all projects in the folder             │
│  └── Useful for department/team boundaries                   │
│                                                                │
│  Billing-account-level sink                                    │
│  └── Routes billing-related logs                              │
│                                                                │
│  Project-level sink                                            │
│  ├── Routes logs from a single project                       │
│  └── Most common sink scope                                  │
│                                                                │
│  Note: Org/folder sinks can only route logs that originate   │
│  from child resources. They use "includeChildren=true".      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Key IAM Roles

| Role | Description |
|------|-------------|
| `roles/logging.viewer` | Read logs in Log Explorer |
| `roles/logging.privateLogViewer` | Read logs including Data Access audit logs |
| `roles/logging.logWriter` | Write log entries |
| `roles/logging.configWriter` | Create/update sinks, exclusions, buckets |
| `roles/logging.admin` | Full control over Logging resources |
| `roles/logging.bucketWriter` | Write to a specific log bucket (used by sink service accounts) |
| `roles/logging.viewAccessor` | Access logs through a log view |

---

## Part 3 — Log Entry Structure

### Anatomy of a Log Entry

```
┌──────────────────────────────────────────────────────────────┐
│                  LOG ENTRY STRUCTURE                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  {                                                             │
│    "logName":     "projects/my-proj/logs/cloudaudit%2F...",   │
│    "resource": {                                               │
│      "type": "gce_instance",                                  │
│      "labels": {                                               │
│        "instance_id": "123456",                                │
│        "zone": "us-central1-a",                               │
│        "project_id": "my-proj"                                │
│      }                                                        │
│    },                                                          │
│    "timestamp":   "2026-05-17T10:30:00.000Z",                │
│    "receiveTimestamp": "2026-05-17T10:30:00.123Z",           │
│    "severity":    "ERROR",                                    │
│    "insertId":    "abc123",                                   │
│    "labels": {                                                │
│      "env": "prod",                                           │
│      "version": "v2.1.0"                                     │
│    },                                                          │
│    "textPayload": "Connection refused to database",          │
│    // OR                                                      │
│    "jsonPayload": { ... structured data ... },               │
│    // OR                                                      │
│    "protoPayload": { ... audit log data ... },               │
│    "httpRequest": { ... },    // if HTTP-related             │
│    "operation":   { ... },    // long-running operation      │
│    "trace":       "projects/my-proj/traces/TRACE_ID",        │
│    "spanId":      "SPAN_ID",                                 │
│    "sourceLocation": {                                        │
│      "file": "app.py",                                       │
│      "line": 42,                                              │
│      "function": "handle_request"                             │
│    }                                                           │
│  }                                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Severity Levels

| Level | Value | Description |
|-------|-------|-------------|
| `DEFAULT` | 0 | No severity specified |
| `DEBUG` | 100 | Debug information |
| `INFO` | 200 | Informational |
| `NOTICE` | 300 | Normal but significant |
| `WARNING` | 400 | Warning conditions |
| `ERROR` | 500 | Error conditions |
| `CRITICAL` | 600 | Critical conditions |
| `ALERT` | 700 | Immediate action needed |
| `EMERGENCY` | 800 | System unusable |

### Payload Types

| Payload | Description | Typical Source |
|---------|-------------|----------------|
| `textPayload` | Plain-text message | Simple application logs |
| `jsonPayload` | Structured JSON object | Structured application logs, Cloud Run |
| `protoPayload` | Protocol buffer (audit data) | Audit Logs (Admin Activity, Data Access) |

### Special JSON Fields

When writing structured logs with `jsonPayload`, certain fields are automatically promoted to top-level `LogEntry` fields:

| JSON Field | Maps To | Description |
|------------|---------|-------------|
| `severity` or `level` | `LogEntry.severity` | Log severity level |
| `message` or `msg` | `LogEntry.textPayload` summary | Displayed in Log Explorer |
| `time` or `timestamp` | `LogEntry.timestamp` | Entry timestamp |
| `httpRequest` | `LogEntry.httpRequest` | HTTP request details |
| `logging.googleapis.com/trace` | `LogEntry.trace` | Trace ID for correlation |
| `logging.googleapis.com/spanId` | `LogEntry.spanId` | Span ID |
| `logging.googleapis.com/labels` | `LogEntry.labels` | User labels |
| `logging.googleapis.com/sourceLocation` | `LogEntry.sourceLocation` | Source file/line |
| `logging.googleapis.com/operation` | `LogEntry.operation` | Long-running operation |

---

## Part 4 — Log Explorer

### Using Log Explorer

Log Explorer is the primary interface for searching and viewing logs:

```
┌──────────────────────────────────────────────────────────────┐
│                    LOG EXPLORER                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Console → Logging → Log Explorer                             │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Resource: [All resources              ▼]            │    │
│  │  Log name: [All logs                   ▼]            │    │
│  │  Severity: [≥ WARNING                  ▼]            │    │
│  │                                                      │    │
│  │  Query: [                                           ]│    │
│  │  resource.type="cloud_run_revision"                  │    │
│  │  severity>=ERROR                                     │    │
│  │  textPayload:"connection refused"                    │    │
│  │                                                      │    │
│  │  [Run Query]  [Stream logs]  [Save]  [Last 1 hour ▼]│    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Histogram (log volume over time)                     │    │
│  │  ▃▅▇█▆▃▂▁▂▃▅▇█▇▅▃▂▁▁▂▃▅▇▇▅▃▂                      │    │
│  │  10:00   10:15   10:30   10:45   11:00               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Log Entries                                          │    │
│  │  ──────────────────────────────────────────────────── │    │
│  │  ▸ 10:32:15  ERROR  cloud_run_revision               │    │
│  │    Connection refused to 10.0.1.5:5432               │    │
│  │  ▸ 10:31:42  ERROR  cloud_run_revision               │    │
│  │    Timeout waiting for database response             │    │
│  │  ▸ 10:30:05  WARNING cloud_run_revision              │    │
│  │    Retry attempt 3 of 5 for database query           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Actions: [Create alert] [Create sink] [Create metric]       │
│           [Download] [Copy link]                              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Logging Query Language

```bash
# ─── Basic Filters ──────────────────────────────────────────
# By resource type
resource.type = "cloud_run_revision"

# By severity
severity >= ERROR

# By log name
logName = "projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"

# By text content (substring match)
textPayload : "connection refused"

# By JSON field
jsonPayload.statusCode = 500

# By label
labels.env = "production"

# ─── Combining Filters ──────────────────────────────────────
# AND (implicit — newline or explicit AND)
resource.type = "gce_instance"
severity >= WARNING

# OR
severity = ERROR OR severity = CRITICAL

# NOT
severity != DEBUG
NOT textPayload : "healthcheck"

# ─── Time-Based ─────────────────────────────────────────────
# Specific time range
timestamp >= "2026-05-17T10:00:00Z"
timestamp <= "2026-05-17T11:00:00Z"

# ─── Resource Labels ────────────────────────────────────────
resource.labels.instance_id = "1234567890"
resource.labels.zone = "us-central1-a"
resource.labels.cluster_name = "prod-cluster"
resource.labels.namespace_name = "default"

# ─── HTTP Request Fields ────────────────────────────────────
httpRequest.requestMethod = "POST"
httpRequest.status = 500
httpRequest.requestUrl : "/api/checkout"

# ─── Audit Log Queries ──────────────────────────────────────
# Who deleted a resource?
protoPayload.methodName = "compute.instances.delete"
protoPayload.authenticationInfo.principalEmail = "user@example.com"

# ─── Regex ───────────────────────────────────────────────────
textPayload =~ "error.*timeout"
jsonPayload.message =~ "failed to connect to .+"

# ─── Complex Example ────────────────────────────────────────
resource.type = "cloud_run_revision"
resource.labels.service_name = "checkout-api"
severity >= ERROR
NOT textPayload : "healthcheck"
timestamp >= "2026-05-17T09:00:00Z"
```

### Log Explorer Features

| Feature | Description |
|---------|-------------|
| **Query builder** | UI dropdowns + text query |
| **Log streaming** | Real-time tail (Stream logs button) |
| **Histogram** | Visual log volume over time |
| **Field explorer** | Sidebar showing top fields and values |
| **Expand entry** | Show full JSON / proto payload |
| **Similar entries** | Group similar log lines |
| **Pin fields** | Pin important fields as columns |
| **Saved queries** | Save and reuse frequent queries |
| **Share link** | Copy query URL for team sharing |
| **Download** | Export results as CSV or JSON |

---

## Part 5 — Log Router & Sinks

### Creating Sinks

```bash
# ─── Sink to BigQuery ────────────────────────────────────────
gcloud logging sinks create bq-all-errors \
    bigquery.googleapis.com/projects/my-project/datasets/error_logs \
    --log-filter='severity >= ERROR' \
    --use-partitioned-tables \
    --description="All error logs to BigQuery"

# ─── Sink to Cloud Storage ──────────────────────────────────
gcloud logging sinks create gcs-archive \
    storage.googleapis.com/my-log-archive-bucket \
    --log-filter='resource.type = "gce_instance"' \
    --description="Archive VM logs to GCS"

# ─── Sink to Pub/Sub ────────────────────────────────────────
gcloud logging sinks create pubsub-security \
    pubsub.googleapis.com/projects/my-project/topics/security-logs \
    --log-filter='logName : "cloudaudit.googleapis.com"' \
    --description="Stream audit logs to Pub/Sub"

# ─── Sink to another Log Bucket ─────────────────────────────
gcloud logging sinks create bucket-prod-logs \
    logging.googleapis.com/projects/my-project/locations/us-central1/buckets/prod-logs \
    --log-filter='labels.env = "production"' \
    --description="Production logs to dedicated bucket"

# ─── Sink to another project's bucket ───────────────────────
gcloud logging sinks create cross-project-sink \
    logging.googleapis.com/projects/central-logging/locations/global/buckets/all-projects \
    --log-filter='severity >= WARNING' \
    --description="Centralise warnings and above"
```

### Sink Service Account Permissions

When you create a sink, Cloud Logging creates a unique writer identity (service account). You must grant this identity permission on the destination:

```bash
# Get the sink's writer identity
gcloud logging sinks describe bq-all-errors --format='value(writerIdentity)'
# Output: serviceAccount:p123456789-123456@gcp-sa-logging.iam.gserviceaccount.com

# Grant BigQuery Data Editor to the sink's SA
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:p123456789-123456@gcp-sa-logging.iam.gserviceaccount.com" \
    --role="roles/bigquery.dataEditor"

# Grant Storage Object Creator for GCS sinks
gcloud storage buckets add-iam-policy-binding gs://my-log-archive-bucket \
    --member="serviceAccount:p123456789-123456@gcp-sa-logging.iam.gserviceaccount.com" \
    --role="roles/storage.objectCreator"

# Grant Pub/Sub Publisher for Pub/Sub sinks
gcloud pubsub topics add-iam-policy-binding security-logs \
    --member="serviceAccount:p123456789-123456@gcp-sa-logging.iam.gserviceaccount.com" \
    --role="roles/pubsub.publisher"
```

### Organization-Level Aggregated Sinks

```bash
# Create org-level aggregated sink (all child projects)
gcloud logging sinks create org-audit-sink \
    bigquery.googleapis.com/projects/security-project/datasets/org_audit \
    --organization=ORGANIZATION_ID \
    --include-children \
    --log-filter='logName : "cloudaudit.googleapis.com%2Factivity"' \
    --description="All Admin Activity logs across the org"

# Folder-level aggregated sink
gcloud logging sinks create folder-audit-sink \
    storage.googleapis.com/dept-log-archive \
    --folder=FOLDER_ID \
    --include-children \
    --log-filter='severity >= ERROR'
```

### Managing Sinks

```bash
# List sinks
gcloud logging sinks list

# Describe a sink
gcloud logging sinks describe bq-all-errors

# Update sink filter
gcloud logging sinks update bq-all-errors \
    --log-filter='severity >= WARNING'

# Disable the _Default sink (stop storing logs in _Default bucket)
gcloud logging sinks update _Default --disabled

# Delete a sink
gcloud logging sinks delete bq-all-errors
```

### Sink Destination Comparison

| Destination | Latency | Best For | Query Capability |
|-------------|---------|----------|-----------------|
| **Log Bucket** | Real-time | Standard log viewing, Log Explorer | Logging query language |
| **BigQuery** | Near real-time (streaming) | Ad-hoc SQL analytics, long-term analysis | SQL |
| **Cloud Storage** | Batched (hourly) | Archival, compliance, cheapest long-term | Must load into another tool |
| **Pub/Sub** | Real-time | Streaming to external systems (Splunk, Datadog), custom processing | N/A (consumer processes) |

---

## Part 6 — Exclusion Filters

### What Are Exclusion Filters?

Exclusion filters let you discard log entries before they are stored, reducing ingestion costs without affecting routing to other sinks:

```
┌──────────────────────────────────────────────────────────────┐
│               EXCLUSION FILTERS                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Log Entry → Log Router                                       │
│                 │                                              │
│                 ├── _Required sink → _Required bucket          │
│                 │   (CANNOT be excluded)                       │
│                 │                                              │
│                 ├── Exclusion filter check                     │
│                 │   ┌─────────────────────────┐               │
│                 │   │ Match: health check logs │               │
│                 │   │ → DROPPED (not stored,   │               │
│                 │   │   not charged)           │               │
│                 │   └─────────────────────────┘               │
│                 │                                              │
│                 ├── _Default sink → _Default bucket            │
│                 │   (only non-excluded entries)                │
│                 │                                              │
│                 └── User sinks                                 │
│                     (exclusions apply to _Default only;       │
│                      user sinks with matching filters         │
│                      still receive entries)                    │
│                                                                │
│  Important:                                                    │
│  • Exclusion filters only affect the _Default sink            │
│  • User-created sinks are NOT affected by exclusions          │
│  • You can add exclusion filters on individual sinks too      │
│  • Excluded logs are not stored and not charged               │
│  • Percentage-based sampling is supported                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating Exclusion Filters

```bash
# Exclude health check logs from _Default bucket
gcloud logging sinks update _Default \
    --add-exclusion=name=no-healthchecks,filter='httpRequest.requestUrl="/health" OR httpRequest.requestUrl="/readiness"'

# Exclude debug logs in production
gcloud logging sinks update _Default \
    --add-exclusion=name=no-debug,filter='severity = DEBUG'

# Exclude high-volume GKE data-plane logs
gcloud logging sinks update _Default \
    --add-exclusion=name=no-gke-dataplane,filter='resource.type="k8s_container" labels."k8s-pod/app"="istio-proxy"'

# Sample only 10% of access logs (exclude 90%)
gcloud logging sinks update _Default \
    --add-exclusion=name=sample-access-logs,filter='resource.type="http_load_balancer"',disabled=false \
    --description="Keep 10% of LB access logs"
# Note: Use sample() function in filter for percentage-based sampling
# filter: 'resource.type="http_load_balancer" AND sample(insertId, 0.1)'
# Keeps 10%, so exclusion filter inverts: exclude the 90%
# filter: 'resource.type="http_load_balancer" AND NOT sample(insertId, 0.1)'

# Exclusion on a specific user sink
gcloud logging sinks update my-bq-sink \
    --add-exclusion=name=no-info,filter='severity <= INFO'

# List exclusion filters on _Default
gcloud logging sinks describe _Default --format="yaml(exclusions)"

# Remove an exclusion filter
gcloud logging sinks update _Default \
    --remove-exclusions=no-debug
```

### Common Exclusion Patterns

| Pattern | Filter | Estimated Savings |
|---------|--------|-------------------|
| Health checks | `httpRequest.requestUrl="/health"` | High (very frequent) |
| Load balancer access logs | `resource.type="http_load_balancer"` | Very high |
| Debug severity | `severity = DEBUG` | Medium |
| Istio/Envoy sidecar | `resource.type="k8s_container" labels."k8s-pod/app"="istio-proxy"` | Very high |
| GKE system logs | `resource.type="k8s_container" resource.labels.namespace_name="kube-system"` | High |
| Successful requests only | `httpRequest.status < 400` | High |
| Specific noisy service | `resource.labels.service_name="noisy-svc"` | Varies |

---

## Part 7 — Log Buckets & Views

### Log Buckets

```
┌──────────────────────────────────────────────────────────────┐
│                  LOG BUCKETS                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Built-in buckets (per project):                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  _Required bucket                                    │     │
│  │  • Admin Activity, System Event, Access Transparency │     │
│  │  • 400-day retention (fixed)                         │     │
│  │  • Cannot be deleted or modified                     │     │
│  │  • Location: global                                  │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  _Default bucket                                     │     │
│  │  • All other logs (from _Default sink)               │     │
│  │  • 30-day retention (configurable 1–3,650 days)      │     │
│  │  • Can be upgraded to Log Analytics                  │     │
│  │  • Location: global                                  │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                                │
│  Custom buckets:                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  prod-logs (us-central1)                             │     │
│  │  • Retention: 90 days                                │     │
│  │  • Locked: retention cannot be reduced               │     │
│  │  • Log Analytics enabled                             │     │
│  │  • CMEK encryption                                   │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  compliance-logs (eu)                                │     │
│  │  • Retention: 2,555 days (7 years)                   │     │
│  │  • Locked: true                                      │     │
│  │  • Region: EU for data residency                     │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Managing Buckets

```bash
# Create custom bucket
gcloud logging buckets create prod-logs \
    --location=us-central1 \
    --retention-days=90 \
    --description="Production application logs" \
    --enable-analytics

# Create locked bucket (retention cannot be reduced once set)
gcloud logging buckets create compliance-logs \
    --location=europe-west1 \
    --retention-days=2555 \
    --locked \
    --description="Compliance logs — 7 year retention"

# Update _Default bucket retention
gcloud logging buckets update _Default \
    --location=global \
    --retention-days=60

# Enable Log Analytics on existing bucket
gcloud logging buckets update _Default \
    --location=global \
    --enable-analytics

# List buckets
gcloud logging buckets list --location=-

# Describe bucket
gcloud logging buckets describe prod-logs --location=us-central1

# Delete custom bucket (must be empty / past retention)
gcloud logging buckets delete prod-logs --location=us-central1
```

### Log Views

Log views control which logs within a bucket a user can access:

```
┌──────────────────────────────────────────────────────────────┐
│                    LOG VIEWS                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Bucket: prod-logs                                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  View: _AllLogs (built-in)                            │    │
│  │  Filter: (none — shows everything)                    │    │
│  │  Access: logging.admin, logging.viewer                │    │
│  ├──────────────────────────────────────────────────────┤    │
│  │  View: app-team-view                                  │    │
│  │  Filter: resource.type="cloud_run_revision"           │    │
│  │  Access: app-team@example.com → logging.viewAccessor │    │
│  ├──────────────────────────────────────────────────────┤    │
│  │  View: security-view                                  │    │
│  │  Filter: logName:"cloudaudit.googleapis.com"          │    │
│  │  Access: security@example.com → logging.viewAccessor │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Use case: Multi-tenant bucket where different teams          │
│  should only see their own logs                               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create a log view
gcloud logging views create app-team-view \
    --bucket=prod-logs \
    --location=us-central1 \
    --log-filter='resource.type="cloud_run_revision"' \
    --description="Cloud Run logs for app team"

# Grant access to the view
gcloud logging views add-iam-policy-binding app-team-view \
    --bucket=prod-logs \
    --location=us-central1 \
    --member="group:app-team@example.com" \
    --role="roles/logging.viewAccessor"

# List views
gcloud logging views list --bucket=prod-logs --location=us-central1

# Delete view
gcloud logging views delete app-team-view \
    --bucket=prod-logs --location=us-central1
```

---

## Part 8 — Log-Based Metrics

### What Are Log-Based Metrics?

Log-based metrics convert log entries into Cloud Monitoring metrics, letting you chart and alert on log patterns:

```
┌──────────────────────────────────────────────────────────────┐
│              LOG-BASED METRICS                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Log entries  ──→  Log-based metric  ──→  Cloud Monitoring   │
│                                                                │
│  Types:                                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Counter metric                                       │    │
│  │  • Counts log entries matching a filter               │    │
│  │  • Example: count of 5xx errors per minute           │    │
│  ├──────────────────────────────────────────────────────┤    │
│  │  Distribution metric                                  │    │
│  │  • Extracts a numeric value from log entries         │    │
│  │  • Creates a distribution (histogram)                │    │
│  │  • Example: response latency distribution            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  System-defined log-based metrics:                             │
│  • logging.googleapis.com/byte_count                          │
│  • logging.googleapis.com/log_entry_count                     │
│  • logging.googleapis.com/exports/byte_count                  │
│  • logging.googleapis.com/exports/error_count                 │
│                                                                │
│  User-defined log-based metrics:                               │
│  • logging.googleapis.com/user/YOUR_METRIC_NAME               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating Log-Based Metrics

```bash
# Counter metric — count 5xx errors
gcloud logging metrics create http_5xx_errors \
    --description="Count of HTTP 5xx error log entries" \
    --log-filter='resource.type="cloud_run_revision" httpRequest.status>=500'

# Counter metric with labels
gcloud logging metrics create api_errors_by_method \
    --description="API errors by HTTP method" \
    --log-filter='resource.type="cloud_run_revision" severity>=ERROR' \
    --label-extractors='method=EXTRACT(httpRequest.requestMethod),service=EXTRACT(resource.labels.service_name)'

# Distribution metric — response latency from logs
gcloud logging metrics create response_latency \
    --description="Response latency from access logs" \
    --log-filter='resource.type="cloud_run_revision" httpRequest.latency!=""' \
    --field-name="httpRequest.latency" \
    --bucket-boundaries="0.01,0.05,0.1,0.25,0.5,1.0,2.5,5.0,10.0"

# List metrics
gcloud logging metrics list

# Describe metric
gcloud logging metrics describe http_5xx_errors

# Update metric filter
gcloud logging metrics update http_5xx_errors \
    --log-filter='resource.type="cloud_run_revision" httpRequest.status>=500 NOT httpRequest.requestUrl="/internal"'

# Delete metric
gcloud logging metrics delete http_5xx_errors
```

### Using Log-Based Metrics in Monitoring

```
Metric type format in Cloud Monitoring:
  logging.googleapis.com/user/http_5xx_errors

Use in:
  • Metrics Explorer (chart the metric)
  • Dashboard widgets
  • Alerting policy conditions
  • MQL queries
```

```bash
# Example: MQL query using a log-based metric
# fetch global
# | metric 'logging.googleapis.com/user/http_5xx_errors'
# | align rate(5m)
# | group_by [metric.service], sum(val())
```

---

## Part 9 — Log-Based Alerts

### Direct Log-Based Alerts

Log-based alerts fire when a log entry matches a specific filter, without needing an intermediate metric:

```
┌──────────────────────────────────────────────────────────────┐
│              LOG-BASED ALERTS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Log entry matches filter                                      │
│       │                                                        │
│       ▼                                                        │
│  ┌────────────────────────────────────────────┐              │
│  │  Alerting Policy (log-based condition)      │              │
│  │                                            │              │
│  │  Filter: severity = CRITICAL               │              │
│  │          resource.type = "gce_instance"     │              │
│  │                                            │              │
│  │  Notification period: 5 minutes             │              │
│  │  (avoid noisy repeated alerts)             │              │
│  │                                            │              │
│  │  Auto-close: 7 days                         │              │
│  └─────────────────┬──────────────────────────┘              │
│                    │                                          │
│                    ▼                                          │
│  ┌────────────────────────────────────────────┐              │
│  │  Notification Channels                      │              │
│  │  • PagerDuty (critical)                     │              │
│  │  • Slack #incidents                         │              │
│  └────────────────────────────────────────────┘              │
│                                                                │
│  Key differences from metric-based alerts:                    │
│  • No aggregation / alignment (fires on single entry)        │
│  • Cannot use "for duration" (threshold-over-time)           │
│  • Has notification rate limit (min 5 min between)           │
│  • Simpler but less flexible                                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating Log-Based Alerts via CLI

```bash
# Create log-based alert policy
gcloud alpha monitoring policies create \
    --policy-from-file=log-alert-policy.json
```

### Log-Based Alert Policy JSON

```json
{
  "displayName": "Critical Error Detected",
  "documentation": {
    "content": "A CRITICAL severity log entry was detected.\n\nCheck the logs in Log Explorer for details.",
    "mimeType": "text/markdown"
  },
  "conditions": [
    {
      "displayName": "Critical log entry",
      "conditionMatchedLog": {
        "filter": "severity = CRITICAL AND resource.type = \"cloud_run_revision\"",
        "labelExtractors": {
          "service": "EXTRACT(resource.labels.service_name)"
        }
      }
    }
  ],
  "combiner": "OR",
  "notificationChannels": [
    "projects/my-project/notificationChannels/CHANNEL_ID"
  ],
  "alertStrategy": {
    "notificationRateLimit": {
      "period": "300s"
    },
    "autoClose": "604800s"
  }
}
```

### Log-Based Alert vs Metric-Based Alert

| Feature | Log-Based Alert | Metric-Based Alert (from log-based metric) |
|---------|----------------|---------------------------------------------|
| Trigger | Single log entry match | Metric crosses threshold over time |
| Aggregation | None | Full alignment + reduction |
| Duration / window | No (immediate) | Yes (e.g., "for 5 minutes") |
| Rate limit | Min 5 min between notifications | Based on condition duration |
| Use case | Rare critical events | Rate/count thresholds |
| Example | "OOM Killed" log appeared | Error rate > 5% for 5 min |

---

## Part 10 — Log Analytics

### What Is Log Analytics?

Log Analytics lets you query logs using SQL (BigQuery dialect) directly in the Cloud Console. It requires upgrading a log bucket to be "Analytics-enabled":

```
┌──────────────────────────────────────────────────────────────┐
│                 LOG ANALYTICS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Regular Log Bucket                  Analytics-Enabled Bucket │
│  ┌─────────────────────┐            ┌─────────────────────┐ │
│  │ Log Explorer         │            │ Log Explorer         │ │
│  │ (query language)     │            │ (query language)     │ │
│  │                     │    ──→     │ + SQL Queries        │ │
│  │                     │  upgrade   │ + BigQuery linked    │ │
│  │                     │            │   dataset (optional) │ │
│  └─────────────────────┘            └─────────────────────┘ │
│                                                                │
│  SQL Query Example:                                            │
│  SELECT                                                        │
│    timestamp,                                                  │
│    severity,                                                   │
│    json_payload.message AS message,                           │
│    resource.labels.service_name AS service                    │
│  FROM                                                          │
│    `my-project.global._Default._AllLogs`                      │
│  WHERE                                                         │
│    severity = 'ERROR'                                         │
│    AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)│
│  ORDER BY timestamp DESC                                      │
│  LIMIT 100                                                     │
│                                                                │
│  Benefits:                                                     │
│  • JOIN logs with BigQuery tables                             │
│  • Aggregations (COUNT, AVG, GROUP BY)                        │
│  • Window functions (trending, ranking)                       │
│  • Regular expressions                                         │
│  • Share queries across teams                                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Enabling Log Analytics

```bash
# Enable Analytics on _Default bucket
gcloud logging buckets update _Default \
    --location=global \
    --enable-analytics

# Create a new bucket with analytics
gcloud logging buckets create analytics-bucket \
    --location=us-central1 \
    --retention-days=90 \
    --enable-analytics

# Create a BigQuery linked dataset (optional — for querying from BigQuery UI)
gcloud logging links create my-link \
    --bucket=_Default \
    --location=global
```

### Log Analytics SQL Examples

```sql
-- Top error messages in the last hour
SELECT
  JSON_VALUE(json_payload.message) AS error_message,
  COUNT(*) AS occurrences
FROM
  `my-project.global._Default._AllLogs`
WHERE
  severity = 'ERROR'
  AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
GROUP BY error_message
ORDER BY occurrences DESC
LIMIT 20;

-- Request latency percentiles by service
SELECT
  resource.labels.service_name AS service,
  APPROX_QUANTILES(
    CAST(JSON_VALUE(http_request.latency) AS FLOAT64), 100
  )[OFFSET(50)] AS p50_latency,
  APPROX_QUANTILES(
    CAST(JSON_VALUE(http_request.latency) AS FLOAT64), 100
  )[OFFSET(95)] AS p95_latency,
  APPROX_QUANTILES(
    CAST(JSON_VALUE(http_request.latency) AS FLOAT64), 100
  )[OFFSET(99)] AS p99_latency
FROM
  `my-project.global._Default._AllLogs`
WHERE
  http_request IS NOT NULL
  AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
GROUP BY service;

-- Error trends over time (hourly buckets)
SELECT
  TIMESTAMP_TRUNC(timestamp, HOUR) AS hour,
  severity,
  COUNT(*) AS count
FROM
  `my-project.global._Default._AllLogs`
WHERE
  severity IN ('ERROR', 'CRITICAL')
  AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY hour, severity
ORDER BY hour DESC;

-- JOIN logs with BigQuery metadata table
SELECT
  l.timestamp,
  l.json_payload,
  m.team_owner,
  m.cost_center
FROM
  `my-project.global._Default._AllLogs` AS l
JOIN
  `my-project.metadata.service_owners` AS m
ON
  resource.labels.service_name = m.service_name
WHERE
  l.severity = 'ERROR';
```

---

## Part 11 — Audit Logs

### Audit Log Types

```
┌──────────────────────────────────────────────────────────────┐
│                   AUDIT LOGS                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Admin Activity Audit Logs                              │  │
│  │  • API calls that modify resources (create, delete,     │  │
│  │    update, set IAM policy)                              │  │
│  │  • Always enabled, cannot be disabled                   │  │
│  │  • Free — stored 400 days in _Required bucket           │  │
│  │  • Log: cloudaudit.googleapis.com/activity              │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Data Access Audit Logs                                 │  │
│  │  • API calls that read resource data or metadata        │  │
│  │  • Disabled by default (can be very high volume)        │  │
│  │  • Charged at standard ingestion rate                   │  │
│  │  • Log: cloudaudit.googleapis.com/data_access           │  │
│  │  • Subtypes: ADMIN_READ, DATA_READ, DATA_WRITE         │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  System Event Audit Logs                                │  │
│  │  • Google-initiated actions (live migration,            │  │
│  │    auto-scaling, maintenance)                           │  │
│  │  • Always enabled, cannot be disabled                   │  │
│  │  • Free — stored 400 days in _Required bucket           │  │
│  │  • Log: cloudaudit.googleapis.com/system_event          │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Policy Denied Audit Logs                               │  │
│  │  • Logged when a request is denied due to policy        │  │
│  │    (VPC Service Controls, IAM deny policies)            │  │
│  │  • Always enabled, cannot be disabled                   │  │
│  │  • Free — stored 400 days in _Required bucket           │  │
│  │  • Log: cloudaudit.googleapis.com/policy                │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Enabling Data Access Audit Logs

```bash
# Enable Data Access logs for a specific service (BigQuery)
# via gcloud (set audit config in IAM policy)

# Get current policy
gcloud projects get-iam-policy my-project --format=json > policy.json

# Edit policy.json to add auditConfigs:
```

```json
{
  "auditConfigs": [
    {
      "service": "bigquery.googleapis.com",
      "auditLogConfigs": [
        { "logType": "ADMIN_READ" },
        { "logType": "DATA_READ" },
        { "logType": "DATA_WRITE" }
      ]
    },
    {
      "service": "storage.googleapis.com",
      "auditLogConfigs": [
        { "logType": "DATA_READ" },
        { "logType": "DATA_WRITE" }
      ]
    },
    {
      "service": "allServices",
      "auditLogConfigs": [
        { "logType": "ADMIN_READ" }
      ]
    }
  ]
}
```

```bash
# Apply updated policy
gcloud projects set-iam-policy my-project policy.json
```

### Querying Audit Logs

```bash
# Who deleted a VM instance?
resource.type = "gce_instance"
logName = "projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"
protoPayload.methodName = "v1.compute.instances.delete"

# Who changed IAM policy?
logName = "projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"
protoPayload.methodName = "SetIamPolicy"

# Failed permission checks
logName = "projects/my-project/logs/cloudaudit.googleapis.com%2Fpolicy"

# BigQuery data access (who queried what)
logName = "projects/my-project/logs/cloudaudit.googleapis.com%2Fdata_access"
protoPayload.serviceName = "bigquery.googleapis.com"
protoPayload.methodName = "jobservice.jobcompleted"

# All actions by a specific user
protoPayload.authenticationInfo.principalEmail = "user@example.com"

# Service account key creation
protoPayload.methodName = "google.iam.admin.v1.CreateServiceAccountKey"
```

### Audit Log Entry Structure

```json
{
  "protoPayload": {
    "@type": "type.googleapis.com/google.cloud.audit.AuditLog",
    "serviceName": "compute.googleapis.com",
    "methodName": "v1.compute.instances.delete",
    "authenticationInfo": {
      "principalEmail": "admin@example.com"
    },
    "authorizationInfo": [
      {
        "permission": "compute.instances.delete",
        "granted": true,
        "resourceAttributes": {
          "name": "projects/my-project/zones/us-central1-a/instances/my-vm"
        }
      }
    ],
    "requestMetadata": {
      "callerIp": "203.0.113.1",
      "callerSuppliedUserAgent": "google-cloud-sdk gcloud/450.0.0"
    },
    "request": { },
    "response": { },
    "resourceName": "projects/my-project/zones/us-central1-a/instances/my-vm"
  },
  "resource": {
    "type": "gce_instance",
    "labels": { }
  },
  "severity": "NOTICE",
  "logName": "projects/my-project/logs/cloudaudit.googleapis.com%2Factivity",
  "timestamp": "2026-05-17T10:30:00Z"
}
```

---

## Part 12 — Application & Access Logs

### GCP Service Logs

| Service | Log Type | Auto-Enabled | Log Name |
|---------|----------|-------------|----------|
| Cloud Run | Request logs | Yes | `run.googleapis.com/requests` |
| Cloud Run | Container stdout/stderr | Yes | `run.googleapis.com/stdout`, `stderr` |
| App Engine | Request logs | Yes | `appengine.googleapis.com/request_log` |
| Cloud Functions | Execution logs | Yes | `cloudfunctions.googleapis.com/cloud-functions` |
| GKE | Container logs | Yes | `stdout`, `stderr` |
| GKE | System component logs | Yes | Various kube-system logs |
| Compute Engine | Serial port | Yes | `compute.googleapis.com/serial_port_output` |
| Cloud SQL | Database logs | Configurable | `cloudsql.googleapis.com/...` |
| VPC Flow Logs | Network flows | Configurable | `compute.googleapis.com/vpc_flows` |
| Firewall Rules | Allowed/denied | Configurable | `compute.googleapis.com/firewall` |
| Load Balancer | Access logs | Configurable | `requests` (under LB resource) |
| Cloud DNS | Query logs | Configurable | `dns.googleapis.com/dns_queries` |

### Writing Application Logs

```python
# ─── Python — google-cloud-logging library ──────────────────
import google.cloud.logging

client = google.cloud.logging.Client()
client.setup_logging()  # Redirects Python logging to Cloud Logging

import logging
logger = logging.getLogger(__name__)

logger.info("Application started")
logger.error("Failed to connect to database", extra={
    "json_fields": {
        "database": "orders-db",
        "retry_count": 3,
    }
})

# ─── Python — structured logging (recommended for Cloud Run) ─
import json
import sys

def log_structured(severity, message, **kwargs):
    entry = {"severity": severity, "message": message, **kwargs}
    print(json.dumps(entry), file=sys.stderr)

log_structured("ERROR", "Database connection failed",
    database="orders-db",
    retry_count=3,
    **{"logging.googleapis.com/trace": f"projects/my-project/traces/{trace_id}"}
)
```

```go
// Go — structured logging for Cloud Run / GKE
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type LogEntry struct {
    Severity string `json:"severity"`
    Message  string `json:"message"`
    Component string `json:"component,omitempty"`
    Trace    string `json:"logging.googleapis.com/trace,omitempty"`
}

func logInfo(msg string) {
    entry := LogEntry{Severity: "INFO", Message: msg}
    data, _ := json.Marshal(entry)
    fmt.Fprintln(os.Stdout, string(data))
}

func logError(msg string, component string) {
    entry := LogEntry{Severity: "ERROR", Message: msg, Component: component}
    data, _ := json.Marshal(entry)
    fmt.Fprintln(os.Stderr, string(data))
}
```

```javascript
// Node.js — structured logging
const log = (severity, message, fields = {}) => {
  const entry = { severity, message, ...fields };
  console.log(JSON.stringify(entry));
};

log('INFO', 'Request processed', {
  requestId: '123',
  latencyMs: 45,
  'logging.googleapis.com/trace': `projects/my-project/traces/${traceId}`,
});

log('ERROR', 'Payment failed', {
  orderId: 'ORD-456',
  error: 'Insufficient funds',
});
```

### VPC Flow Logs

```bash
# Enable VPC Flow Logs on a subnet
gcloud compute networks subnets update my-subnet \
    --region=us-central1 \
    --enable-flow-logs \
    --logging-aggregation-interval=INTERVAL_5_SEC \
    --logging-flow-sampling=0.5 \
    --logging-metadata=INCLUDE_ALL_METADATA

# Query flow logs
# resource.type="gce_subnetwork"
# logName="projects/my-project/logs/compute.googleapis.com%2Fvpc_flows"
# jsonPayload.connection.dest_ip="10.0.1.5"
```

---

## Part 13 — Ops Agent & Structured Logging

### Ops Agent Overview

```
┌──────────────────────────────────────────────────────────────┐
│                  OPS AGENT                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  The Ops Agent is the primary agent for Compute Engine VMs.   │
│  It collects both logs and metrics.                            │
│                                                                │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Ops Agent                                          │      │
│  │  ├── Logging (Fluent Bit-based)                     │      │
│  │  │   ├── System logs (/var/log/syslog, messages)   │      │
│  │  │   ├── Application logs (custom file paths)      │      │
│  │  │   └── Third-party (nginx, apache, mysql, etc.)  │      │
│  │  │                                                  │      │
│  │  └── Metrics (OpenTelemetry Collector-based)        │      │
│  │      ├── Host metrics (CPU, memory, disk, network) │      │
│  │      ├── Third-party (nginx, mysql, redis, etc.)   │      │
│  │      └── Custom (StatsD, OTLP, JMX, etc.)         │      │
│  └────────────────────────────────────────────────────┘      │
│                                                                │
│  vs Legacy Logging Agent (deprecated):                        │
│  • Legacy = fluentd-based (higher resource usage)             │
│  • Ops Agent = fluent-bit + OTel (lighter, recommended)       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Installing the Ops Agent

```bash
# Install on a single VM
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Install via Terraform (startup script)
# Or use OS Config / VM Manager agent policies for fleet deployment

# Verify agent is running
sudo systemctl status google-cloud-ops-agent

# Restart agent after config changes
sudo systemctl restart google-cloud-ops-agent
```

### Ops Agent Configuration

```yaml
# /etc/google-cloud-ops-agent/config.yaml

logging:
  receivers:
    # Built-in system log receiver (enabled by default)
    syslog:
      type: files
      include_paths:
        - /var/log/messages
        - /var/log/syslog
    
    # Custom application logs
    app_logs:
      type: files
      include_paths:
        - /var/log/my-app/*.log
      record_log_name: my-app

    # Nginx access logs (built-in parser)
    nginx_access:
      type: nginx_access
      include_paths:
        - /var/log/nginx/access.log

    # Nginx error logs
    nginx_error:
      type: nginx_error
      include_paths:
        - /var/log/nginx/error.log

    # JSON-formatted application logs
    json_app:
      type: files
      include_paths:
        - /var/log/my-app/structured.json
      record_log_name: my-app-structured

  processors:
    # Parse JSON logs
    parse_json:
      type: parse_json
      time_key: timestamp
      time_format: "%Y-%m-%dT%H:%M:%S.%LZ"

  service:
    pipelines:
      default_pipeline:
        receivers: [syslog]
      app_pipeline:
        receivers: [app_logs]
      nginx_pipeline:
        receivers: [nginx_access, nginx_error]
      json_pipeline:
        receivers: [json_app]
        processors: [parse_json]

metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
    
    nginx:
      type: nginx
      stub_status_url: http://localhost:80/nginx_status
      collection_interval: 60s

    mysql:
      type: mysql
      endpoint: localhost:3306
      username: monitoring
      password: "${MYSQL_MONITORING_PASSWORD}"
      collection_interval: 60s

  service:
    pipelines:
      default_pipeline:
        receivers: [hostmetrics]
      nginx_pipeline:
        receivers: [nginx]
      mysql_pipeline:
        receivers: [mysql]
```

### Structured Logging Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│         STRUCTURED LOGGING BEST PRACTICES                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Always use JSON format                                    │
│     ✗ print("Error: failed to connect")                      │
│     ✓ {"severity":"ERROR","message":"failed to connect",     │
│        "component":"db","retry":3}                            │
│                                                                │
│  2. Include trace context for correlation                     │
│     Add "logging.googleapis.com/trace" field to correlate    │
│     logs with Cloud Trace spans                               │
│                                                                │
│  3. Use standard severity levels                              │
│     DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL             │
│                                                                │
│  4. Include request context                                   │
│     requestId, userId (hashed), path, method                 │
│                                                                │
│  5. Avoid logging sensitive data                              │
│     No passwords, tokens, PII in log entries                 │
│     Use Data Loss Prevention (DLP) API to redact if needed   │
│                                                                │
│  6. Set appropriate severity                                  │
│     Don't log everything as ERROR                             │
│     Reserve CRITICAL for system-breaking issues               │
│                                                                │
│  7. Use labels for dimensions                                 │
│     environment, version, region, team                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 14: Console Walkthrough — Log Sinks, Buckets & Metrics

### Creating a Log Sink (Log Router)

```
Console → Logging → Log Router → CREATE SINK

┌─────────────────────────────────────────────────────────────────┐
│           CREATE SINK (Step 1 of 4)                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Sink details ──                                               │
│ Sink name: [error-logs-to-bigquery]                             │
│ Sink description: [Route error logs to BigQuery for analysis]  │
│                                                                   │
│                                   [NEXT]                        │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           Step 2: Sink destination                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Sink service:                                                   │
│ ○ Cloud Logging bucket                                         │
│ ● BigQuery dataset                                              │
│ ○ Cloud Storage bucket                                          │
│ ○ Pub/Sub topic                                                  │
│ ○ Splunk                                                         │
│ ○ Other project (Cloud Logging bucket in another project)      │
│                                                                   │
│ BigQuery dataset:                                               │
│ ● Select an existing dataset: [error_logs ▼]                   │
│ ○ Create a new dataset                                         │
│                                                                   │
│ ☑ Use partitioned tables                                        │
│ ⚡ Recommended for cost/query performance.                       │
│                                                                   │
│                                   [NEXT]                        │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           Step 3: Choose logs to include in sink                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Inclusion filter (what to route):                               │
│ [severity >= ERROR]                                             │
│ ⚡ Uses Cloud Logging query syntax.                              │
│   Examples:                                                     │
│   severity >= ERROR                                             │
│   resource.type = "gce_instance"                                │
│   resource.type = "k8s_container" AND severity >= WARNING      │
│   logName = "projects/my-proj/logs/my-app"                     │
│                                                                   │
│                                   [NEXT]                        │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           Step 4: Choose logs to exclude from sink (optional)     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Exclusion filter name: [exclude-debug]                          │
│ Exclusion filter: [severity = DEBUG]                            │
│ ⚡ Logs matching exclusion filter are NOT routed to this sink.   │
│                                                                   │
│                          [CREATE SINK]                          │
│                                                                   │
│ ⚡ After creation, a service account is auto-generated.          │
│   You must grant it write access to the destination:           │
│   ├── BigQuery: BigQuery Data Editor role                       │
│   ├── GCS: Storage Object Creator role                         │
│   └── Pub/Sub: Pub/Sub Publisher role                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Log-Based Metric

```
Console → Logging → Log-based Metrics → CREATE METRIC

┌─────────────────────────────────────────────────────────────────┐
│           CREATE LOG-BASED METRIC                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Metric type ──                                                │
│ ● Counter (counts matching log entries)                         │
│ ○ Distribution (extracts numeric value from log field)         │
│                                                                   │
│ ── Details ──                                                     │
│ Log-based metric name: [error-count]                            │
│ Description: [Count of error-level log entries]                 │
│ Units: [1] (default for counter)                                │
│                                                                   │
│ ── Filter selection ──                                           │
│ Build filter:                                                   │
│ [severity >= ERROR]                                             │
│ ⚡ Same query syntax as Log Explorer.                            │
│                                                                   │
│ ── Labels (optional) ──                                          │
│ Label name: [service]                                           │
│ Label type: [STRING ▼]                                          │
│ Field name: [resource.labels.service_name]                      │
│ [+ ADD LABEL]                                                    │
│ ⚡ Labels create metric dimensions for filtering in dashboards. │
│                                                                   │
│                         [CREATE METRIC]                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Log Bucket

```
Console → Logging → Logs Storage → CREATE LOG BUCKET

┌─────────────────────────────────────────────────────────────────┐
│           CREATE LOG BUCKET                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Name ──                                                       │
│ Name: [custom-app-logs]                                         │
│                                                                   │
│ ── Description ──                                                │
│ Description: [Application logs for custom retention]            │
│                                                                   │
│ ── Region ──                                                     │
│ Region: [global ▼]                                              │
│   Options: global, us-central1, europe-west1, asia-south1...   │
│ ⚡ Regional buckets keep logs in that region (data residency).  │
│                                                                   │
│ ── Retention period ──                                           │
│ Retention: [90] days                                            │
│ ⚡ Default _Default bucket: 30 days.                             │
│   Custom buckets: 1–36500 days (10 years).                     │
│   After retention → logs are automatically deleted.            │
│                                                                   │
│ ── Analytics ──                                                   │
│ ☑ Upgrade to use Log Analytics                                  │
│ ⚡ Enables SQL queries on this bucket's logs via BigQuery.      │
│                                                                   │
│ ── Encryption ──                                                 │
│ ● Google-managed encryption key (default)                      │
│ ○ Customer-managed encryption key (CMEK)                       │
│                                                                   │
│                         [CREATE BUCKET]                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Terraform & gcloud CLI Reference

### Terraform — Log Sinks

```hcl
# ─── Project-Level Sink to BigQuery ──────────────────────────
resource "google_logging_project_sink" "bigquery_errors" {
  name        = "bq-error-logs"
  project     = var.project_id
  destination = "bigquery.googleapis.com/projects/${var.project_id}/datasets/${google_bigquery_dataset.logs.dataset_id}"
  filter      = "severity >= ERROR"

  unique_writer_identity = true

  bigquery_options {
    use_partitioned_tables = true
  }
}

# Grant sink SA access to BigQuery dataset
resource "google_bigquery_dataset_iam_member" "sink_writer" {
  dataset_id = google_bigquery_dataset.logs.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = google_logging_project_sink.bigquery_errors.writer_identity
}

# BigQuery dataset for logs
resource "google_bigquery_dataset" "logs" {
  dataset_id = "error_logs"
  project    = var.project_id
  location   = "US"

  default_table_expiration_ms = 7776000000  # 90 days
}

# ─── Sink to Cloud Storage ──────────────────────────────────
resource "google_logging_project_sink" "gcs_archive" {
  name        = "gcs-archive"
  project     = var.project_id
  destination = "storage.googleapis.com/${google_storage_bucket.log_archive.name}"
  filter      = "resource.type = \"gce_instance\""

  unique_writer_identity = true
}

resource "google_storage_bucket" "log_archive" {
  name     = "${var.project_id}-log-archive"
  location = "US"
  project  = var.project_id

  lifecycle_rule {
    condition {
      age = 365
    }
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
  }

  lifecycle_rule {
    condition {
      age = 2555  # 7 years
    }
    action {
      type = "Delete"
    }
  }
}

resource "google_storage_bucket_iam_member" "sink_writer" {
  bucket = google_storage_bucket.log_archive.name
  role   = "roles/storage.objectCreator"
  member = google_logging_project_sink.gcs_archive.writer_identity
}

# ─── Sink to Pub/Sub ────────────────────────────────────────
resource "google_logging_project_sink" "pubsub_audit" {
  name        = "pubsub-audit-logs"
  project     = var.project_id
  destination = "pubsub.googleapis.com/projects/${var.project_id}/topics/${google_pubsub_topic.audit_logs.name}"
  filter      = "logName:\"cloudaudit.googleapis.com\""

  unique_writer_identity = true
}

resource "google_pubsub_topic" "audit_logs" {
  name    = "audit-log-stream"
  project = var.project_id
}

resource "google_pubsub_topic_iam_member" "sink_publisher" {
  topic  = google_pubsub_topic.audit_logs.name
  role   = "roles/pubsub.publisher"
  member = google_logging_project_sink.pubsub_audit.writer_identity
}
```

### Terraform — Org-Level Aggregated Sink

```hcl
resource "google_logging_organization_sink" "org_audit" {
  name             = "org-audit-sink"
  org_id           = var.org_id
  destination      = "bigquery.googleapis.com/projects/${var.security_project}/datasets/${var.audit_dataset}"
  filter           = "logName:\"cloudaudit.googleapis.com%2Factivity\""
  include_children = true

  bigquery_options {
    use_partitioned_tables = true
  }
}
```

### Terraform — Log Bucket & View

```hcl
# Custom log bucket
resource "google_logging_project_bucket_config" "prod_logs" {
  project        = var.project_id
  location       = "us-central1"
  bucket_id      = "prod-logs"
  retention_days = 90
  description    = "Production application logs"

  enable_analytics = true
}

# Sink routing to the custom bucket
resource "google_logging_project_sink" "to_prod_bucket" {
  name        = "prod-log-sink"
  project     = var.project_id
  destination = "logging.googleapis.com/projects/${var.project_id}/locations/us-central1/buckets/prod-logs"
  filter      = "labels.env = \"production\""

  unique_writer_identity = true
}

# Log view for app team
resource "google_logging_log_view" "app_team" {
  name        = "app-team-view"
  bucket      = google_logging_project_bucket_config.prod_logs.id
  location    = "us-central1"
  filter      = "resource.type = \"cloud_run_revision\""
  description = "Cloud Run logs for app team"
}
```

### Terraform — Exclusion Filters

```hcl
# Project-level exclusion
resource "google_logging_project_exclusion" "no_healthchecks" {
  name        = "no-healthcheck-logs"
  project     = var.project_id
  description = "Exclude health check access logs"
  filter      = "httpRequest.requestUrl = \"/health\" OR httpRequest.requestUrl = \"/readiness\""
}

# Project-level exclusion — debug logs
resource "google_logging_project_exclusion" "no_debug" {
  name        = "no-debug-logs"
  project     = var.project_id
  description = "Exclude debug severity logs in production"
  filter      = "severity = DEBUG"
}

# Exclusion on a specific sink
resource "google_logging_project_sink" "default_modified" {
  name        = "_Default"
  project     = var.project_id
  destination = "logging.googleapis.com/projects/${var.project_id}/locations/global/buckets/_Default"

  exclusions {
    name   = "no-lb-access-logs"
    filter = "resource.type = \"http_load_balancer\" AND httpRequest.status < 400"
  }

  exclusions {
    name   = "no-istio-proxy"
    filter = "resource.type = \"k8s_container\" AND labels.\"k8s-pod/app\" = \"istio-proxy\""
  }
}

# Organization-level exclusion
resource "google_logging_organization_exclusion" "org_no_debug" {
  name        = "org-no-debug"
  org_id      = var.org_id
  description = "Exclude debug logs org-wide"
  filter      = "severity = DEBUG"
}
```

### Terraform — Log-Based Metric

```hcl
resource "google_logging_metric" "error_count" {
  name    = "http_5xx_errors"
  project = var.project_id
  filter  = "resource.type = \"cloud_run_revision\" AND httpRequest.status >= 500"

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
    unit        = "1"

    labels {
      key         = "service"
      value_type  = "STRING"
      description = "Cloud Run service name"
    }
  }

  label_extractors = {
    "service" = "EXTRACT(resource.labels.service_name)"
  }
}

# Distribution metric
resource "google_logging_metric" "response_latency" {
  name    = "response_latency_distribution"
  project = var.project_id
  filter  = "resource.type = \"cloud_run_revision\" AND httpRequest.latency != \"\""

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "DISTRIBUTION"
    unit        = "s"
  }

  value_extractor = "EXTRACT(httpRequest.latency)"

  bucket_options {
    explicit_buckets {
      bounds = [0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
    }
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# SINKS
# ═══════════════════════════════════════════════════════════════
# Create sink
gcloud logging sinks create SINK_NAME DESTINATION \
    --log-filter='FILTER' \
    --description="DESCRIPTION"

# List sinks
gcloud logging sinks list

# Describe sink
gcloud logging sinks describe SINK_NAME

# Update sink
gcloud logging sinks update SINK_NAME \
    --log-filter='NEW_FILTER'

# Delete sink
gcloud logging sinks delete SINK_NAME

# Disable _Default sink
gcloud logging sinks update _Default --disabled

# Add exclusion to sink
gcloud logging sinks update SINK_NAME \
    --add-exclusion=name=EXCLUSION_NAME,filter='FILTER'

# Remove exclusion from sink
gcloud logging sinks update SINK_NAME \
    --remove-exclusions=EXCLUSION_NAME

# ═══════════════════════════════════════════════════════════════
# BUCKETS
# ═══════════════════════════════════════════════════════════════
# Create bucket
gcloud logging buckets create BUCKET_ID \
    --location=LOCATION \
    --retention-days=DAYS \
    --enable-analytics

# List buckets
gcloud logging buckets list --location=-

# Update bucket retention
gcloud logging buckets update BUCKET_ID \
    --location=LOCATION \
    --retention-days=NEW_DAYS

# Delete bucket
gcloud logging buckets delete BUCKET_ID --location=LOCATION

# ═══════════════════════════════════════════════════════════════
# VIEWS
# ═══════════════════════════════════════════════════════════════
# Create view
gcloud logging views create VIEW_ID \
    --bucket=BUCKET_ID \
    --location=LOCATION \
    --log-filter='FILTER'

# List views
gcloud logging views list --bucket=BUCKET_ID --location=LOCATION

# Delete view
gcloud logging views delete VIEW_ID \
    --bucket=BUCKET_ID --location=LOCATION

# ═══════════════════════════════════════════════════════════════
# LOG-BASED METRICS
# ═══════════════════════════════════════════════════════════════
# Create counter metric
gcloud logging metrics create METRIC_NAME \
    --description="DESCRIPTION" \
    --log-filter='FILTER'

# List metrics
gcloud logging metrics list

# Update metric
gcloud logging metrics update METRIC_NAME \
    --log-filter='NEW_FILTER'

# Delete metric
gcloud logging metrics delete METRIC_NAME

# ═══════════════════════════════════════════════════════════════
# READING / WRITING LOGS
# ═══════════════════════════════════════════════════════════════
# Read logs
gcloud logging read 'severity >= ERROR' \
    --limit=50 \
    --format=json

# Read with time range
gcloud logging read 'resource.type="cloud_run_revision" severity>=WARNING' \
    --freshness=1h \
    --limit=100

# Write a log entry
gcloud logging write my-log "Test log entry" \
    --severity=INFO \
    --payload-type=text

# Write structured log entry
gcloud logging write my-log '{"message":"Test","code":200}' \
    --severity=INFO \
    --payload-type=json

# ═══════════════════════════════════════════════════════════════
# EXCLUSIONS (project-level)
# ═══════════════════════════════════════════════════════════════
# Create exclusion
gcloud logging exclusions create EXCLUSION_NAME \
    --log-filter='FILTER' \
    --description="DESCRIPTION"

# List exclusions
gcloud logging exclusions list

# Update exclusion
gcloud logging exclusions update EXCLUSION_NAME \
    --log-filter='NEW_FILTER'

# Delete exclusion
gcloud logging exclusions delete EXCLUSION_NAME

# ═══════════════════════════════════════════════════════════════
# LOG ANALYTICS LINKS
# ═══════════════════════════════════════════════════════════════
# Create BigQuery linked dataset
gcloud logging links create LINK_ID \
    --bucket=BUCKET_ID \
    --location=LOCATION

# List links
gcloud logging links list --bucket=BUCKET_ID --location=LOCATION

# Delete link
gcloud logging links delete LINK_ID \
    --bucket=BUCKET_ID --location=LOCATION

# ═══════════════════════════════════════════════════════════════
# TAIL (LIVE STREAMING)
# ═══════════════════════════════════════════════════════════════
# Live tail logs
gcloud logging tail 'resource.type="cloud_run_revision" severity>=ERROR'

# Tail with buffer window
gcloud logging tail 'severity>=WARNING' --buffer-window=30s
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Centralized Logging with Cost Optimization

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: CENTRALIZED LOGGING WITH COST CONTROL                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Problem: Multiple projects generating 500 GB/day of logs.            │
│  Cost without optimization: ~$225/day ($6,750/month).                 │
│  Target: Retain critical logs, archive rest, cut costs 60%+.          │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Project A    Project B    Project C    Project D              │   │
│  │       \           |           |           /                   │   │
│  │        └──────────┼───────────┼──────────┘                    │   │
│  │                   ▼                                            │   │
│  │  ┌─────────────────────────────────────────────────────┐     │   │
│  │  │              Log Router                              │     │   │
│  │  │                                                      │     │   │
│  │  │  Exclusions (applied first):                         │     │   │
│  │  │  ✗ Health check logs           (~30% volume)        │     │   │
│  │  │  ✗ DEBUG severity              (~15% volume)        │     │   │
│  │  │  ✗ Istio proxy logs            (~20% volume)        │     │   │
│  │  │  ✗ LB 2xx access logs (keep 10% sample)  (~25%)    │     │   │
│  │  │                                                      │     │   │
│  │  │  After exclusions: ~60 GB/day (88% reduction!)      │     │   │
│  │  │                                                      │     │   │
│  │  │  Sinks:                                              │     │   │
│  │  │  ├→ _Default bucket (30d): WARNING+ (10 GB/day)     │     │   │
│  │  │  ├→ BigQuery dataset: ERROR+ (2 GB/day, analytics)  │     │   │
│  │  │  ├→ Cloud Storage: ALL (60 GB/day, 7yr archive)     │     │   │
│  │  │  └→ Pub/Sub: Audit logs (→ SIEM)                    │     │   │
│  │  └─────────────────────────────────────────────────────┘     │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  Cost after optimization:                                              │
│  • Ingestion: 60 GB × $0.50 = $30/day (+ 50 GB free)                 │
│  • Storage (GCS): $0.02/GB = pennies/day                              │
│  • Total: ~$900/month (87% savings)                                   │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```hcl
# Exclusion filters
resource "google_logging_project_exclusion" "healthchecks" {
  name    = "exclude-healthchecks"
  project = var.project_id
  filter  = <<-EOT
    httpRequest.requestUrl="/health"
    OR httpRequest.requestUrl="/readiness"
    OR httpRequest.requestUrl="/liveness"
  EOT
}

resource "google_logging_project_exclusion" "debug" {
  name    = "exclude-debug"
  project = var.project_id
  filter  = "severity = DEBUG"
}

resource "google_logging_project_exclusion" "istio_proxy" {
  name    = "exclude-istio-proxy"
  project = var.project_id
  filter  = "resource.type = \"k8s_container\" AND labels.\"k8s-pod/app\" = \"istio-proxy\""
}

resource "google_logging_project_exclusion" "lb_sampling" {
  name    = "sample-lb-access-logs"
  project = var.project_id
  filter  = "resource.type = \"http_load_balancer\" AND NOT sample(insertId, 0.1)"
}

# Sink to BigQuery for analytics (errors only)
resource "google_logging_project_sink" "bq_errors" {
  name                   = "bq-errors"
  project                = var.project_id
  destination            = "bigquery.googleapis.com/projects/${var.project_id}/datasets/error_analytics"
  filter                 = "severity >= ERROR"
  unique_writer_identity = true

  bigquery_options {
    use_partitioned_tables = true
  }
}

# Sink to Cloud Storage for long-term archive
resource "google_logging_project_sink" "gcs_all" {
  name                   = "gcs-archive-all"
  project                = var.project_id
  destination            = "storage.googleapis.com/${google_storage_bucket.logs.name}"
  unique_writer_identity = true
}

# Sink to Pub/Sub for SIEM (audit logs)
resource "google_logging_project_sink" "siem_audit" {
  name                   = "pubsub-siem"
  project                = var.project_id
  destination            = "pubsub.googleapis.com/projects/${var.project_id}/topics/siem-audit"
  filter                 = "logName:\"cloudaudit.googleapis.com\""
  unique_writer_identity = true
}
```

### Pattern 2: Security Audit Trail with Org-Level Sinks

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: ORGANIZATION-WIDE SECURITY AUDIT TRAIL                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Requirement: Capture all admin activity and security events          │
│  across 50+ projects for compliance (SOC 2, PCI DSS, HIPAA).         │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Organization: example.com                                │        │
│  │  ├── Folder: Production                                   │        │
│  │  │   ├── Project: prod-api                                │        │
│  │  │   ├── Project: prod-data                               │        │
│  │  │   └── Project: prod-infra                              │        │
│  │  ├── Folder: Staging                                      │        │
│  │  │   └── ...                                              │        │
│  │  └── Folder: Development                                  │        │
│  │      └── ...                                              │        │
│  └──────────────────────────────────────────────────────────┘        │
│                   │                                                    │
│                   ▼                                                    │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Org-Level Aggregated Sinks                               │        │
│  │                                                           │        │
│  │  Sink 1: audit-to-bq (includeChildren=true)              │        │
│  │  Filter: logName:"cloudaudit.googleapis.com%2Factivity"   │        │
│  │  Dest: BigQuery (security-project.org_audit_logs)         │        │
│  │  → For security analytics, compliance queries             │        │
│  │                                                           │        │
│  │  Sink 2: audit-to-pubsub (includeChildren=true)          │        │
│  │  Filter: logName:"cloudaudit.googleapis.com"              │        │
│  │  Dest: Pub/Sub → Cloud Function → SIEM (Splunk/Chronicle)│        │
│  │  → For real-time threat detection                        │        │
│  │                                                           │        │
│  │  Sink 3: all-to-gcs (includeChildren=true)               │        │
│  │  Filter: (none — captures everything)                     │        │
│  │  Dest: Cloud Storage (locked bucket, 7yr retention)       │        │
│  │  → For regulatory compliance archive                      │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Alert examples (from BigQuery analytics):                             │
│  • IAM policy change on production project                            │
│  • Service account key created                                        │
│  • Firewall rule modified                                             │
│  • VPC network created/deleted                                        │
│  • Bucket made public                                                 │
│  • Suspicious gcloud commands from unknown IP                         │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```hcl
# Org-level sink to BigQuery
resource "google_logging_organization_sink" "audit_to_bq" {
  name             = "org-audit-to-bq"
  org_id           = var.org_id
  destination      = "bigquery.googleapis.com/projects/${var.security_project}/datasets/org_audit_logs"
  filter           = "logName:\"cloudaudit.googleapis.com%2Factivity\""
  include_children = true

  bigquery_options {
    use_partitioned_tables = true
  }
}

# Org-level sink to Pub/Sub for SIEM streaming
resource "google_logging_organization_sink" "audit_to_siem" {
  name             = "org-audit-to-siem"
  org_id           = var.org_id
  destination      = "pubsub.googleapis.com/projects/${var.security_project}/topics/siem-ingest"
  filter           = "logName:\"cloudaudit.googleapis.com\""
  include_children = true
}

# Org-level sink to GCS for compliance archive
resource "google_logging_organization_sink" "all_to_gcs" {
  name             = "org-all-to-archive"
  org_id           = var.org_id
  destination      = "storage.googleapis.com/${google_storage_bucket.compliance_archive.name}"
  include_children = true
}

resource "google_storage_bucket" "compliance_archive" {
  name     = "${var.org_name}-compliance-logs"
  location = "US"
  project  = var.security_project

  retention_policy {
    retention_period = 220752000  # 7 years in seconds
    is_locked        = true
  }

  lifecycle_rule {
    condition { age = 90 }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }

  lifecycle_rule {
    condition { age = 365 }
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
  }

  lifecycle_rule {
    condition { age = 730 }
    action {
      type          = "SetStorageClass"
      storage_class = "ARCHIVE"
    }
  }
}

# Log-based alert: service account key created
resource "google_monitoring_alert_policy" "sa_key_created" {
  display_name = "[Security] Service Account Key Created"
  project      = var.security_project
  combiner     = "OR"

  conditions {
    display_name = "SA key creation detected"
    condition_matched_log {
      filter = "protoPayload.methodName = \"google.iam.admin.v1.CreateServiceAccountKey\""
    }
  }

  notification_channels = [var.security_channel_id]

  alert_strategy {
    notification_rate_limit { period = "300s" }
    auto_close = "604800s"
  }
}
```

### Pattern 3: Application Debugging with Correlated Logs and Traces

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CORRELATED LOGGING + TRACING                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Goal: When an error occurs, see the full request flow across         │
│  services with correlated logs and trace spans.                       │
│                                                                        │
│  Request flow:                                                         │
│  User → LB → API Gateway → Order Service → Payment Service → DB      │
│                                                                        │
│  Each service writes structured logs with trace context:              │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  {"severity": "INFO",                                       │     │
│  │   "message": "Processing order ORD-123",                    │     │
│  │   "orderId": "ORD-123",                                    │     │
│  │   "logging.googleapis.com/trace":                           │     │
│  │     "projects/my-proj/traces/abc123...",                    │     │
│  │   "logging.googleapis.com/spanId": "def456",               │     │
│  │   "logging.googleapis.com/labels": {                        │     │
│  │     "requestId": "req-789"                                  │     │
│  │   }}                                                        │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  In Log Explorer:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  Filter by trace: trace="projects/my-proj/traces/abc123"    │     │
│  │                                                              │     │
│  │  10:30:00.100  INFO  api-gateway    Received POST /orders  │     │
│  │  10:30:00.150  INFO  order-service  Processing ORD-123     │     │
│  │  10:30:00.200  INFO  payment-svc    Charging $99.00        │     │
│  │  10:30:00.350  ERROR payment-svc    Payment declined       │     │
│  │  10:30:00.360  ERROR order-service  Order failed: payment  │     │
│  │  10:30:00.370  ERROR api-gateway    Returning 402          │     │
│  │                                                              │     │
│  │  [View in Cloud Trace] ← click to see waterfall           │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  In Cloud Trace:                                                       │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  Trace: abc123...                                           │     │
│  │  ├── api-gateway (270ms)                                    │     │
│  │  │   ├── order-service (210ms)                              │     │
│  │  │   │   ├── payment-service (150ms)  ← ERROR              │     │
│  │  │   │   │   └── db-query (20ms)                            │     │
│  │  │   │   └── (log entries linked here)                      │     │
│  │  │   └── (log entries linked here)                          │     │
│  │  └── (log entries linked here)                              │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation (Cloud Run + Python):**

```python
# Structured logging with trace correlation for Cloud Run
import json
import os
import sys

def get_trace_context():
    """Extract trace context from Cloud Run request header."""
    trace_header = os.environ.get("HTTP_X_CLOUD_TRACE_CONTEXT", "")
    if trace_header:
        parts = trace_header.split("/")
        trace_id = parts[0]
        span_id = parts[1].split(";")[0] if len(parts) > 1 else ""
        project = os.environ.get("GOOGLE_CLOUD_PROJECT", "")
        return {
            "logging.googleapis.com/trace": f"projects/{project}/traces/{trace_id}",
            "logging.googleapis.com/spanId": span_id,
        }
    return {}

def log(severity, message, **kwargs):
    """Write a structured log entry with trace correlation."""
    entry = {
        "severity": severity,
        "message": message,
        **get_trace_context(),
        **kwargs,
    }
    print(json.dumps(entry), flush=True)

# Usage in request handler
def handle_order(request):
    order_id = request.json.get("orderId")
    log("INFO", f"Processing order {order_id}", orderId=order_id)
    
    try:
        result = process_payment(order_id)
        log("INFO", f"Order {order_id} completed", orderId=order_id, amount=result["amount"])
        return {"status": "success"}, 200
    except PaymentError as e:
        log("ERROR", f"Payment failed for {order_id}", orderId=order_id, error=str(e))
        return {"status": "failed", "error": str(e)}, 402
```

---

## Quick Reference

| Action | Command / Location |
|--------|-------------------|
| View logs | Console → Logging → Log Explorer |
| Read logs (CLI) | `gcloud logging read 'FILTER' --limit=N` |
| Write log entry | `gcloud logging write LOG_NAME "MESSAGE" --severity=LEVEL` |
| Stream logs live | `gcloud logging tail 'FILTER'` |
| Create sink | `gcloud logging sinks create NAME DESTINATION --log-filter='F'` |
| List sinks | `gcloud logging sinks list` |
| Create exclusion | `gcloud logging exclusions create NAME --log-filter='F'` |
| Create bucket | `gcloud logging buckets create ID --location=LOC --retention-days=N` |
| Create log view | `gcloud logging views create ID --bucket=B --location=L --log-filter='F'` |
| Create log metric | `gcloud logging metrics create NAME --log-filter='F'` |
| Enable analytics | `gcloud logging buckets update _Default --location=global --enable-analytics` |
| Disable _Default sink | `gcloud logging sinks update _Default --disabled` |
| Query audit logs | Filter: `logName:"cloudaudit.googleapis.com%2Factivity"` |
| Check sink identity | `gcloud logging sinks describe NAME --format='value(writerIdentity)'` |

---

## What is Logging? (Beginner Explanation)

### The Diary Analogy

Think of **logs as a diary your applications write automatically**. Every time something happens — a user logs in, a request fails, a database connection drops — your application writes a timestamped entry. Cloud Logging collects all these diary entries from every service, stores them, and lets you search through them.

```
┌──────────────────────────────────────────────────────────────┐
│            LOGGING = YOUR APPLICATION'S DIARY                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Personal Diary              Application Logs                 │
│  ──────────────              ────────────────                 │
│                                                                │
│  "May 22, 9:00 AM           "2026-05-22T09:00:00Z INFO        │
│   Woke up, had coffee"       Server started on port 8080"     │
│                                                                │
│  "May 22, 9:15 AM           "2026-05-22T09:15:00Z INFO        │
│   Left for work"             User alice@co.com logged in"     │
│                                                                │
│  "May 22, 10:30 AM          "2026-05-22T10:30:00Z ERROR       │
│   Spilled coffee on shirt"   Connection refused to DB:5432"   │
│                                                                │
│  "May 22, 11:00 AM          "2026-05-22T11:00:00Z WARNING     │
│   Boss looked angry"         Memory usage at 85%, nearing     │
│                               threshold"                      │
│                                                                │
│  Your diary helps you         Logs help engineers              │
│  remember what happened       figure out what happened         │
│  and when.                    and why.                         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Why Do Logs Matter?

| Reason | Explanation | Example |
|--------|------------|----------|
| **Debugging** | When something breaks, logs tell you *what* happened and *when* | "The payment service started returning 500 errors at 10:32 AM because the database connection pool was exhausted" |
| **Security** | Logs record who did what — critical for detecting suspicious activity | "An unknown IP tried to login as admin 500 times in 1 minute" (brute-force attack) |
| **Compliance** | Many regulations (HIPAA, PCI-DSS, SOX) require you to keep audit logs for years | "We can prove that only authorised users accessed patient records" |
| **Performance** | Logs reveal slow queries, timeout patterns, and bottlenecks | "The /search endpoint takes 5s on average — the SQL query needs an index" |
| **Incident response** | During an outage, logs are the first thing engineers check | "Let me filter logs from the last 30 minutes for severity=ERROR" |

### Structured vs Unstructured Logs

```
┌──────────────────────────────────────────────────────────────┐
│         STRUCTURED vs UNSTRUCTURED LOGS                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Unstructured Log (plain text):                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 2026-05-22 10:30:00 ERROR Failed to process order      │  │
│  │ 12345 for user alice@co.com: payment gateway timeout   │  │
│  └────────────────────────────────────────────────────────┘  │
│  ❌ Hard to search — "Find all orders for alice" requires    │
│     text parsing and regex                                    │
│  ❌ Inconsistent format across services                       │
│  ❌ Cannot aggregate or group easily                          │
│                                                                │
│  Structured Log (JSON):                                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ {                                                       │  │
│  │   "timestamp": "2026-05-22T10:30:00Z",                 │  │
│  │   "severity": "ERROR",                                  │  │
│  │   "message": "Failed to process order",                 │  │
│  │   "orderId": "12345",                                   │  │
│  │   "user": "alice@co.com",                               │  │
│  │   "error": "payment gateway timeout",                   │  │
│  │   "service": "checkout-api",                            │  │
│  │   "latencyMs": 30000                                    │  │
│  │ }                                                       │  │
│  └────────────────────────────────────────────────────────┘  │
│  ✅ Easy to search — jsonPayload.user = "alice@co.com"       │
│  ✅ Consistent fields across services                         │
│  ✅ Can aggregate — "count of errors by service per hour"     │
│  ✅ Cloud Logging auto-parses JSON fields                     │
│                                                                │
│  Bottom line: Always prefer structured (JSON) logs.           │
│  Cloud Logging parses JSON automatically and makes every      │
│  field searchable.                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

> **One-liner**: If monitoring tells you "the patient's heart rate spiked", logging tells you "the patient ate three pizzas at 2 AM" — it gives you the *story* behind the numbers.

---

## Console Walkthrough: Managing & Deleting

### Deleting Log Sinks

```
┌──────────────────────────────────────────────────────────────┐
│         DELETE LOG SINKS FROM CONSOLE                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Logging → Log Router                               │
│                                                                │
│  Step 2: Find the sink                                         │
│  • You'll see all sinks listed: _Required, _Default, and      │
│    any user-created sinks                                     │
│  • Use the filter bar to search by sink name                  │
│                                                                │
│  Step 3: Delete                                                │
│  • Click the ⋮ (three-dot menu) on the right side of the     │
│    sink row                                                   │
│  • Select "Delete sink"                                       │
│  • Confirm the deletion                                       │
│                                                                │
│  ⚠ You CANNOT delete the _Required or _Default sinks.         │
│    You can only disable _Default (toggle it off) or add       │
│    exclusion filters to it.                                    │
│                                                                │
│  ⚠ Deleting a sink stops log routing to that destination.     │
│    Logs already delivered are NOT deleted from the             │
│    destination (BigQuery, GCS, etc.).                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud logging sinks list
gcloud logging sinks delete SINK_NAME
```

### Deleting Log Buckets

```
┌──────────────────────────────────────────────────────────────┐
│         DELETE LOG BUCKETS FROM CONSOLE                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Logging → Log Storage                              │
│                                                                │
│  Step 2: Find the bucket                                       │
│  • You'll see _Required, _Default, and custom buckets         │
│  • Each shows location, retention period, and analytics       │
│    status                                                     │
│                                                                │
│  Step 3: Delete                                                │
│  • Click the ⋮ (three-dot menu) next to the custom bucket     │
│  • Select "Delete bucket"                                     │
│  • Confirm the deletion                                       │
│                                                                │
│  ⚠ You CANNOT delete _Required or _Default buckets.           │
│    Only user-created custom buckets can be deleted.            │
│                                                                │
│  ⚠ If a locked bucket still has logs within its retention     │
│    period, it cannot be deleted. You must wait until the       │
│    retention period expires.                                   │
│                                                                │
│  ⚠ Delete all sinks routing to this bucket FIRST, otherwise   │
│    those sinks will start producing export errors.             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud logging buckets list --location=-
gcloud logging buckets delete BUCKET_ID --location=LOCATION
```

### Managing Exclusion Filters

```
┌──────────────────────────────────────────────────────────────┐
│         MANAGE EXCLUSION FILTERS FROM CONSOLE                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Logging → Log Router                               │
│                                                                │
│  Step 2: Find the sink with exclusion filters                  │
│  • Exclusion filters are attached to sinks                    │
│  • Click on the sink name (typically _Default) to view        │
│    its exclusions                                             │
│                                                                │
│  Step 3: View / Edit / Delete exclusions                       │
│  • Click the ⋮ (three-dot menu) on the sink → "Edit sink"    │
│  • Scroll to the "Choose logs to filter out of sink"          │
│    section (exclusion filters)                                │
│  • Each exclusion shows: name, filter, and enabled/disabled   │
│    toggle                                                     │
│  • To DELETE: Click the 🗑 (trash icon) next to the filter    │
│  • To DISABLE temporarily: Toggle the switch off (keeps the   │
│    filter but stops it from excluding logs)                    │
│  • To ADD: Click "Add exclusion" and enter a name + filter    │
│  • Click "Update sink" to save changes                        │
│                                                                │
│  💡 Tip: Disable an exclusion filter (instead of deleting)     │
│     when you want to temporarily see those logs during         │
│     debugging, then re-enable it after.                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
# View exclusions on _Default sink
gcloud logging sinks describe _Default --format="yaml(exclusions)"

# Add exclusion filter
gcloud logging sinks update _Default \
    --add-exclusion=name=no-healthchecks,filter='httpRequest.requestUrl="/health"'

# Remove exclusion filter
gcloud logging sinks update _Default \
    --remove-exclusions=no-healthchecks
```

### Cost Optimization: Excluding High-Volume Logs

Health-check logs are one of the biggest sources of wasted log spend. Load balancers, Kubernetes probes, and uptime checks generate thousands of entries per minute — and they almost never contain useful debugging information.

```
┌──────────────────────────────────────────────────────────────┐
│     REAL-WORLD COST SAVINGS: EXCLUDING HEALTH-CHECK LOGS      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Scenario: E-commerce platform with 20 Cloud Run services    │
│                                                                │
│  BEFORE exclusion filters:                                     │
│  ┌────────────────────────────────────────────────────┐      │
│  │ Log Source                │ Monthly Volume │ Cost   │      │
│  │ Health-check access logs  │ 320 GB         │ $135   │      │
│  │ LB access logs (200 OK)  │ 180 GB         │ $65    │      │
│  │ Application logs          │ 100 GB         │ $25    │      │
│  │ Audit logs (Admin)        │ 2 GB           │ Free   │      │
│  │ ──────────────────────────────────────────────────  │      │
│  │ TOTAL                     │ 602 GB         │ $225   │      │
│  └────────────────────────────────────────────────────┘      │
│                                                                │
│  Exclusion filters applied:                                    │
│  ┌────────────────────────────────────────────────────┐      │
│  │ 1. Exclude health-check logs:                       │      │
│  │    httpRequest.requestUrl="/health" OR               │      │
│  │    httpRequest.requestUrl="/readiness"               │      │
│  │    → Saves 320 GB ($135/month)                      │      │
│  │                                                      │      │
│  │ 2. Sample LB access logs (keep 10%):                │      │
│  │    resource.type="http_load_balancer"                │      │
│  │    AND NOT sample(insertId, 0.1)                    │      │
│  │    → Saves 162 GB ($58/month)                       │      │
│  └────────────────────────────────────────────────────┘      │
│                                                                │
│  AFTER exclusion filters:                                      │
│  ┌────────────────────────────────────────────────────┐      │
│  │ Log Source                │ Monthly Volume │ Cost   │      │
│  │ Health-check access logs  │ 0 GB (excluded)│ $0     │      │
│  │ LB access logs (10%)     │ 18 GB          │ $0*    │      │
│  │ Application logs          │ 100 GB         │ $25    │      │
│  │ Audit logs (Admin)        │ 2 GB           │ Free   │      │
│  │ ──────────────────────────────────────────────────  │      │
│  │ TOTAL                     │ 120 GB         │ $25    │      │
│  │                                                      │      │
│  │ * Within 50 GB free tier                             │      │
│  └────────────────────────────────────────────────────┘      │
│                                                                │
│  💰 Monthly savings: $200/month (~89% reduction)               │
│  💰 Annual savings:  $2,400/year                               │
│                                                                │
│  No useful debugging information was lost — health checks     │
│  always return 200 OK, and 10% LB log sampling is enough      │
│  for traffic analysis.                                         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

> **Takeaway**: Always audit your log volume in the first month. Exclusion filters for health checks and LB access logs are the single biggest cost-saving lever in Cloud Logging.

---

## What's Next?

Continue to **Chapter 38: Cloud Trace** → `38-cloud-trace.md`
