# Chapter 48 — Cloud Scheduler

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Scheduler Fundamentals](#part-1--cloud-scheduler-fundamentals)
- [Part 2: Cron Syntax & Schedule Configuration](#part-2--cron-syntax--schedule-configuration)
- [Part 3: HTTP Targets](#part-3--http-targets)
- [Part 4: Pub/Sub Targets](#part-4--pubsub-targets)
- [Part 5: App Engine Targets](#part-5--app-engine-targets)
- [Part 6: Authentication & Service Accounts](#part-6--authentication--service-accounts)
- [Part 7: Retry Configuration](#part-7--retry-configuration)
- [Part 8: Time Zones & Execution Windows](#part-8--time-zones--execution-windows)
- [Part 9: Job Lifecycle & Management](#part-9--job-lifecycle--management)
- [Part 10: Integration with Other Services](#part-10--integration-with-other-services)
- [Part 11: Monitoring & Troubleshooting](#part-11--monitoring--troubleshooting)
- [Part 12: Terraform & gcloud CLI Reference](#part-12--terraform--gcloud-cli-reference)
- [Part 13: Real-World Patterns](#part-13--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What is Cloud Scheduler? (Beginner Explanation)](#what-is-cloud-scheduler-beginner-explanation)
- [Console Walkthrough: Creating & Managing Jobs](#console-walkthrough-creating--managing-jobs)
- [What's Next?](#whats-next)

---

## Overview

Cloud Scheduler is a fully managed enterprise-grade cron job scheduler. It allows you to schedule virtually any job — including batch, big data, cloud infrastructure operations — using a simple cron or unix-cron format. Cloud Scheduler triggers targets (HTTP endpoints, Pub/Sub topics, or App Engine handlers) on a recurring schedule with built-in retry and authentication support.

---

## Part 1 — Cloud Scheduler Fundamentals

### What Is Cloud Scheduler?

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD SCHEDULER OVERVIEW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Managed cron-as-a-service:                                      │
│                                                                   │
│  ┌──────────────────┐   on schedule    ┌──────────────────┐     │
│  │ Cloud Scheduler  │─────────────────►│ Target           │     │
│  │                  │                   │                  │     │
│  │ "0 2 * * *"      │  HTTP / Pub/Sub  │ Cloud Run        │     │
│  │ (daily at 2am)   │  / App Engine    │ Cloud Functions  │     │
│  │                  │                   │ Pub/Sub Topic    │     │
│  └──────────────────┘                   │ Any HTTP URL     │     │
│                                         └──────────────────┘     │
│                                                                   │
│  Key features:                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ • Cron or unix-cron schedule syntax                      │   │
│  │ • Three target types: HTTP, Pub/Sub, App Engine          │   │
│  │ • Automatic retry with configurable backoff              │   │
│  │ • OIDC/OAuth2 authentication for HTTP targets            │   │
│  │ • Time zone support (IANA time zones)                    │   │
│  │ • Manual job triggering (for testing)                    │   │
│  │ • Job pausing and resuming                               │   │
│  │ • At-least-once execution guarantee                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Cloud Scheduler | AWS EventBridge Scheduler | Azure Logic Apps (Recurrence) |
|---------|--------------------|--------------------------|-----------------------------|
| Service | Cloud Scheduler | EventBridge Scheduler | Logic Apps / Timer Trigger |
| Cron support | Yes | Yes | Yes |
| HTTP target | Yes | Yes (via API dest) | Yes (HTTP action) |
| Message queue target | Yes (Pub/Sub) | Yes (SQS, SNS) | Yes (Service Bus) |
| Auth for HTTP | OIDC / OAuth2 | IAM roles | Managed identity |
| Retry | Yes | Yes | Yes |
| Time zones | Yes (IANA) | Yes (IANA) | Yes |
| Manual trigger | Yes | No | Yes (run trigger) |
| Pricing | 3 free, $0.10/job/month | Free tier + $1/million | Per execution |

### Pricing

| Component | Cost |
|-----------|------|
| First 3 jobs per project | Free |
| Additional jobs | $0.10/job/month |
| Job executions | Free (no per-execution charge) |

---

## Part 2 — Cron Syntax & Schedule Configuration

### Cron Format

```
┌──────────────────────────────────────────────────────────────┐
│         CRON SYNTAX                                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Cloud Scheduler uses unix-cron format:                       │
│                                                                │
│  ┌─── minute (0-59)                                          │
│  │ ┌─── hour (0-23)                                          │
│  │ │ ┌─── day of month (1-31)                                │
│  │ │ │ ┌─── month (1-12)                                     │
│  │ │ │ │ ┌─── day of week (0-6, 0=Sunday)                    │
│  │ │ │ │ │                                                    │
│  * * * * *                                                    │
│                                                                │
│  Special characters:                                           │
│  * = every            , = list             - = range          │
│  / = step                                                     │
│                                                                │
│  Examples:                                                     │
│  ┌────────────────────┬──────────────────────────────────┐   │
│  │ Schedule           │ Meaning                           │   │
│  ├────────────────────┼──────────────────────────────────┤   │
│  │ * * * * *          │ Every minute                      │   │
│  │ 0 * * * *          │ Every hour (on the hour)          │   │
│  │ 0 2 * * *          │ Daily at 2:00 AM                  │   │
│  │ 0 9 * * 1          │ Every Monday at 9:00 AM           │   │
│  │ 0 0 1 * *          │ First day of every month          │   │
│  │ */15 * * * *       │ Every 15 minutes                  │   │
│  │ 0 9-17 * * 1-5     │ Hourly, 9AM-5PM, weekdays        │   │
│  │ 0 0 * * 0          │ Every Sunday at midnight          │   │
│  │ 30 8 1,15 * *      │ 8:30 AM on 1st and 15th          │   │
│  │ 0 */6 * * *        │ Every 6 hours                     │   │
│  └────────────────────┴──────────────────────────────────┘   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 3 — HTTP Targets

### HTTP Target Jobs

```bash
# Create HTTP target job
gcloud scheduler jobs create http daily-report \
    --location=us-central1 \
    --schedule="0 2 * * *" \
    --uri="https://my-service.run.app/generate-report" \
    --http-method=POST \
    --headers="Content-Type=application/json" \
    --message-body='{"type":"daily","format":"pdf"}' \
    --time-zone="America/New_York"

# With OIDC authentication (Cloud Run / Cloud Functions)
gcloud scheduler jobs create http nightly-cleanup \
    --location=us-central1 \
    --schedule="0 3 * * *" \
    --uri="https://my-service.run.app/cleanup" \
    --http-method=POST \
    --oidc-service-account-email=scheduler-sa@my-project.iam.gserviceaccount.com \
    --oidc-token-audience="https://my-service.run.app" \
    --time-zone="UTC"

# With OAuth2 (Google APIs)
gcloud scheduler jobs create http call-google-api \
    --location=us-central1 \
    --schedule="0 */6 * * *" \
    --uri="https://sheets.googleapis.com/v4/spreadsheets/SHEET_ID/values/A1:append" \
    --http-method=POST \
    --oauth-service-account-email=scheduler-sa@my-project.iam.gserviceaccount.com
```

---

## Part 4 — Pub/Sub Targets

### Pub/Sub Target Jobs

```bash
# Create Pub/Sub target job
gcloud scheduler jobs create pubsub hourly-sync \
    --location=us-central1 \
    --schedule="0 * * * *" \
    --topic=sync-topic \
    --message-body='{"action":"sync","source":"scheduler"}' \
    --attributes="triggered_by=scheduler,type=hourly"

# The job publishes a message to the topic on each trigger
# A subscription on the topic processes the message
```

```
┌──────────────────────────────────────────────────────────────┐
│         PUB/SUB TARGET FLOW                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Cloud Scheduler ──on schedule──► Pub/Sub Topic               │
│                    (publish msg)       │                       │
│                                        ├──► Subscription A    │
│                                        │    → Cloud Function  │
│                                        └──► Subscription B    │
│                                             → Cloud Run       │
│                                                                │
│  Advantages over HTTP target:                                  │
│  • Fan-out: one trigger, multiple consumers                  │
│  • Decoupled: consumer can be offline temporarily            │
│  • Pub/Sub handles retry + delivery                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 5 — App Engine Targets

### App Engine Target Jobs

```bash
# Create App Engine target job
gcloud scheduler jobs create app-engine cron-task \
    --location=us-central1 \
    --schedule="*/30 * * * *" \
    --relative-url="/cron/process" \
    --http-method=GET \
    --service=worker \
    --version=v1

# App Engine targets use internal dispatch (no auth needed)
# X-Appengine-Cron: true header is added automatically
```

---

## Part 6 — Authentication & Service Accounts

### Configuring Authentication

```
┌──────────────────────────────────────────────────────────────┐
│         AUTHENTICATION                                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  OIDC Token (for Cloud Run / Cloud Functions / custom):      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Scheduler → includes signed OIDC JWT → target        │    │
│  │  Target validates the JWT automatically               │    │
│  │                                                      │    │
│  │  Required IAM:                                        │    │
│  │  1. SA needs roles/run.invoker on Cloud Run service  │    │
│  │     OR roles/cloudfunctions.invoker for Functions    │    │
│  │  2. Scheduler SA needs iam.serviceAccountUser        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  OAuth2 Token (for Google APIs like Sheets, BigQuery):       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Scheduler → includes OAuth2 access token → API      │    │
│  │  SA needs relevant API scopes / roles                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  No auth (Pub/Sub targets):                                   │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  SA needs roles/pubsub.publisher on the topic        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 7 — Retry Configuration

### Retry Policy

```bash
# Create job with retry config
gcloud scheduler jobs create http resilient-job \
    --location=us-central1 \
    --schedule="0 */4 * * *" \
    --uri="https://my-service.run.app/process" \
    --http-method=POST \
    --max-retry-attempts=5 \
    --min-backoff=5s \
    --max-backoff=3600s \
    --max-doublings=5 \
    --time-zone="UTC"
```

```
Retry behavior:
• HTTP target: retry on 5xx or network error
• Pub/Sub target: retry on publish failure
• App Engine: retry on 5xx or deadline exceeded
• Retries use exponential backoff
• max-retry-attempts=0 means no retries
• Default: 0 (no retries)
```

---

## Part 8 — Time Zones & Execution Windows

### Time Zone Handling

```
┌──────────────────────────────────────────────────────────────┐
│         TIME ZONES                                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Every job has a time zone (IANA format):                     │
│  • America/New_York                                           │
│  • America/Los_Angeles                                        │
│  • Europe/London                                              │
│  • Asia/Kolkata                                               │
│  • UTC (default)                                              │
│                                                                │
│  DST handling:                                                 │
│  • Spring forward: if scheduled time is skipped → runs       │
│    at next valid time                                         │
│  • Fall back: if scheduled time occurs twice → runs once     │
│                                                                │
│  Best practice:                                                │
│  • Use UTC for jobs that must run at exact intervals          │
│  • Use local TZ for business-hour-related jobs               │
│    (e.g., "send report at 9 AM New York time")               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 9 — Job Lifecycle & Management

### Managing Jobs

```bash
# List jobs
gcloud scheduler jobs list --location=us-central1

# Describe job
gcloud scheduler jobs describe daily-report --location=us-central1

# Update job schedule
gcloud scheduler jobs update http daily-report \
    --location=us-central1 \
    --schedule="0 3 * * *"

# Pause job (stops scheduling, keeps config)
gcloud scheduler jobs pause daily-report --location=us-central1

# Resume job
gcloud scheduler jobs resume daily-report --location=us-central1

# Run job manually (for testing)
gcloud scheduler jobs run daily-report --location=us-central1

# Delete job
gcloud scheduler jobs delete daily-report --location=us-central1
```

---

## Part 10 — Integration with Other Services

### Common Integration Patterns

```
┌──────────────────────────────────────────────────────────────┐
│         INTEGRATION PATTERNS                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Scheduler → Cloud Run                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  HTTP target with OIDC auth                          │    │
│  │  Use for: batch processing, report generation        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Scheduler → Cloud Functions                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  HTTP target with OIDC auth                          │    │
│  │  Use for: lightweight scheduled tasks                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Scheduler → Pub/Sub → multiple subscribers                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Pub/Sub target — fan-out to many services           │    │
│  │  Use for: periodic sync, multi-service triggers      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Scheduler → Cloud Tasks (batch task creation)               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Scheduler triggers a function that queries pending   │    │
│  │  items and creates individual Cloud Tasks             │    │
│  │  Use for: periodic batch job → individual tasks      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Scheduler → Workflows                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  HTTP target → Workflows execution API               │    │
│  │  Use for: scheduled multi-step orchestration          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 11 — Monitoring & Troubleshooting

### Key Metrics & Logs

```bash
# View job execution logs
gcloud logging read 'resource.type="cloud_scheduler_job"' \
    --project=my-project --limit=20

# Filter for failed executions
gcloud logging read '
  resource.type="cloud_scheduler_job" AND
  severity>=ERROR
' --project=my-project --limit=10
```

| Issue | Cause | Solution |
|-------|-------|----------|
| Job not triggering | Job is paused | Resume the job |
| PERMISSION_DENIED | SA missing invoke role | Grant `roles/run.invoker` |
| Target returns 5xx | Worker error | Check Cloud Run / Function logs |
| DEADLINE_EXCEEDED | Target too slow | Increase attempt deadline |
| Duplicate executions | At-least-once delivery | Make handler idempotent |

---

## Part 12 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Service Account ─────────────────────────────────────────
resource "google_service_account" "scheduler" {
  account_id   = "scheduler-sa"
  display_name = "Cloud Scheduler Service Account"
  project      = var.project_id
}

# ─── HTTP Target Job ─────────────────────────────────────────
resource "google_cloud_scheduler_job" "daily_report" {
  name        = "daily-report"
  description = "Generate daily sales report"
  schedule    = "0 2 * * *"
  time_zone   = "America/New_York"
  region      = var.region
  project     = var.project_id

  http_target {
    http_method = "POST"
    uri         = "${google_cloud_run_v2_service.report.uri}/generate"
    body        = base64encode(jsonencode({ type = "daily" }))
    headers     = { "Content-Type" = "application/json" }

    oidc_token {
      service_account_email = google_service_account.scheduler.email
      audience              = google_cloud_run_v2_service.report.uri
    }
  }

  retry_config {
    retry_count          = 3
    min_backoff_duration = "5s"
    max_backoff_duration = "3600s"
    max_doublings        = 5
  }
}

# ─── Pub/Sub Target Job ──────────────────────────────────────
resource "google_cloud_scheduler_job" "hourly_sync" {
  name     = "hourly-sync"
  schedule = "0 * * * *"
  region   = var.region
  project  = var.project_id

  pubsub_target {
    topic_name = google_pubsub_topic.sync.id
    data       = base64encode(jsonencode({ action = "sync" }))
    attributes = { triggered_by = "scheduler" }
  }
}

# ─── App Engine Target Job ───────────────────────────────────
resource "google_cloud_scheduler_job" "cron" {
  name     = "app-engine-cron"
  schedule = "*/30 * * * *"
  region   = var.region
  project  = var.project_id

  app_engine_http_target {
    http_method = "GET"
    relative_uri = "/cron/process"
    app_engine_routing {
      service = "worker"
    }
  }
}

# ─── IAM ──────────────────────────────────────────────────────
resource "google_cloud_run_service_iam_member" "scheduler_invoke" {
  project  = var.project_id
  location = var.region
  service  = google_cloud_run_v2_service.report.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.scheduler.email}"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# HTTP JOBS
# ═══════════════════════════════════════════════════════════════
gcloud scheduler jobs create http NAME \
    --location=LOC --schedule="CRON" --uri=URL \
    --http-method=POST --message-body=BODY \
    --oidc-service-account-email=SA \
    --time-zone=TZ --max-retry-attempts=N

# ═══════════════════════════════════════════════════════════════
# PUB/SUB JOBS
# ═══════════════════════════════════════════════════════════════
gcloud scheduler jobs create pubsub NAME \
    --location=LOC --schedule="CRON" --topic=T \
    --message-body=BODY --attributes=K=V

# ═══════════════════════════════════════════════════════════════
# JOB MANAGEMENT
# ═══════════════════════════════════════════════════════════════
gcloud scheduler jobs list --location=LOC
gcloud scheduler jobs describe NAME --location=LOC
gcloud scheduler jobs update http NAME --location=LOC [options]
gcloud scheduler jobs pause NAME --location=LOC
gcloud scheduler jobs resume NAME --location=LOC
gcloud scheduler jobs run NAME --location=LOC    # manual trigger
gcloud scheduler jobs delete NAME --location=LOC
```

---

## Part 13 — Real-World Patterns

### Pattern 1: Nightly Data Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: NIGHTLY DATA PIPELINE                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Cloud Scheduler (2 AM UTC daily)                                    │
│    │                                                                  │
│    ├──► Pub/Sub "pipeline-trigger" topic                             │
│    │         │                                                        │
│    │         ├──► Cloud Function: Extract data from APIs             │
│    │         │    → writes raw data to GCS                           │
│    │         │                                                        │
│    │         ├──► Cloud Function: Export DB to BigQuery               │
│    │         │    → runs BQ load job                                  │
│    │         │                                                        │
│    │         └──► Cloud Run: Generate daily reports                   │
│    │              → emails PDF to stakeholders                       │
│    │                                                                  │
│    │  Single scheduler job triggers entire pipeline via fan-out      │
│    │                                                                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Business-Hours Health Checks

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: BUSINESS-HOURS MONITORING                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Job: "business-health-check"                                        │
│  Schedule: "*/5 9-17 * * 1-5"                                       │
│  Time zone: "America/New_York"                                       │
│  → Every 5 minutes, 9AM-5PM, Monday-Friday                          │
│                                                                        │
│  Target: Cloud Function that checks:                                  │
│  • Payment gateway availability                                     │
│  • Order processing queue depth                                      │
│  • Third-party API response times                                    │
│  → Sends Slack alert if any check fails                              │
│                                                                        │
│  Off-hours: "*/30 * * * 0,6" + "*/30 0-8,18-23 * * 1-5"           │
│  → Less frequent checks on weekends and evenings                     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Scheduled Scaling

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: PREDICTABLE TRAFFIC SCALING                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Scale up before known peak hours, scale down after:                 │
│                                                                        │
│  Job 1: "scale-up"                                                    │
│  Schedule: "0 7 * * 1-5" (7 AM weekdays)                            │
│  → Cloud Function updates Cloud Run min-instances to 10             │
│  → Cloud Function updates MIG min-size to 5                          │
│                                                                        │
│  Job 2: "scale-down"                                                  │
│  Schedule: "0 22 * * 1-5" (10 PM weekdays)                          │
│  → Cloud Function updates Cloud Run min-instances to 1              │
│  → Cloud Function updates MIG min-size to 1                          │
│                                                                        │
│  Job 3: "weekend-minimum"                                             │
│  Schedule: "0 0 * * 6" (Saturday midnight)                           │
│  → Cloud Function sets all to minimum                                │
│                                                                        │
│  Benefits:                                                            │
│  • Pre-warmed instances ready before traffic arrives                 │
│  • Cost savings during off-peak hours                                │
│  • Avoids cold starts during peak                                    │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create HTTP job | `gcloud scheduler jobs create http NAME --schedule="CRON" --uri=URL` |
| Create Pub/Sub job | `... create pubsub NAME --schedule="CRON" --topic=T` |
| OIDC auth | `--oidc-service-account-email=SA` |
| Set time zone | `--time-zone="America/New_York"` |
| Manual trigger | `gcloud scheduler jobs run NAME --location=L` |
| Pause job | `gcloud scheduler jobs pause NAME --location=L` |
| Resume job | `gcloud scheduler jobs resume NAME --location=L` |
| Every minute | `* * * * *` |
| Daily at 2 AM | `0 2 * * *` |
| Weekdays 9-5 hourly | `0 9-17 * * 1-5` |
| Free tier | 3 jobs/project |
| Pricing | $0.10/job/month beyond free tier |

---

## What is Cloud Scheduler? (Beginner Explanation)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD SCHEDULER — THE SIMPLE EXPLANATION                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 💡 Cloud Scheduler is like an alarm clock for your cloud — it      │
│    triggers jobs at specific times, so you don't have to.           │
│                                                                       │
│ Think of a sprinkler system:                                        │
│ ├── You set it to run every morning at 6 AM                        │
│ ├── It turns on automatically — no human needed                    │
│ ├── If something blocks a sprinkler, it tries again                │
│ └── You can pause it, change the schedule, or run it manually      │
│                                                                       │
│ That's exactly what Cloud Scheduler does for your cloud jobs!      │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════════ │
│                                                                       │
│ What is cron? (the scheduling language)                             │
│                                                                       │
│ Cron is a simple format to express "when" something should run.    │
│ It uses 5 fields separated by spaces:                               │
│                                                                       │
│   ┌─── minute (0-59)        "at what minute?"                      │
│   │ ┌─── hour (0-23)        "at what hour?"                        │
│   │ │ ┌─── day of month     "on which day?"                        │
│   │ │ │ ┌─── month (1-12)   "in which month?"                      │
│   │ │ │ │ ┌─── day of week  "on which weekday?"                    │
│   │ │ │ │ │                                                         │
│   * * * * *   ← asterisk (*) means "every"                        │
│                                                                       │
│ Read it right-to-left for plain English:                            │
│   0 9 * * 1   = "At minute 0, hour 9, every day, every month,     │
│                   on Monday" = Every Monday at 9:00 AM             │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════════ │
│                                                                       │
│ Common cron patterns (memorize these!):                             │
│                                                                       │
│ ┌─────────────────────┬─────────────────────────────────────────┐  │
│ │ Cron Expression     │ Plain English                            │  │
│ ├─────────────────────┼─────────────────────────────────────────┤  │
│ │ * * * * *           │ Every minute                             │  │
│ │ */5 * * * *         │ Every 5 minutes                          │  │
│ │ */15 * * * *        │ Every 15 minutes                         │  │
│ │ 0 * * * *           │ Every hour (at :00)                      │  │
│ │ 0 */6 * * *         │ Every 6 hours                            │  │
│ │ 0 9 * * *           │ Daily at 9:00 AM                         │  │
│ │ 0 0 * * *           │ Daily at midnight                        │  │
│ │ 0 2 * * *           │ Daily at 2:00 AM                         │  │
│ │ 0 9 * * 1-5         │ Weekdays at 9:00 AM                      │  │
│ │ 0 9-17 * * 1-5      │ Hourly 9AM-5PM, weekdays only            │  │
│ │ 0 0 * * 0           │ Every Sunday at midnight                 │  │
│ │ 0 0 1 * *           │ First day of every month                 │  │
│ │ 0 0 1 1 *           │ January 1st at midnight (yearly)         │  │
│ │ 30 8 1,15 * *       │ 8:30 AM on the 1st and 15th              │  │
│ └─────────────────────┴─────────────────────────────────────────┘  │
│                                                                       │
│ 💡 Tip: "/" means "every Nth". So */5 = every 5th minute.         │
│ 💡 Tip: "-" means "range". So 1-5 = Monday through Friday.        │
│ 💡 Tip: "," means "list". So 1,15 = 1st and 15th of month.        │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════════ │
│                                                                       │
│ Real-world examples — what do people actually schedule?             │
│                                                                       │
│ ├── Daily database backup at 2 AM                                  │
│ │   → Schedule: "0 2 * * *"                                       │
│ │   → Target: Cloud Function that triggers a DB export             │
│ │                                                                   │
│ ├── Hourly cache refresh                                           │
│ │   → Schedule: "0 * * * *"                                       │
│ │   → Target: Cloud Run service /refresh-cache endpoint            │
│ │                                                                   │
│ ├── Weekly report every Monday at 9 AM                             │
│ │   → Schedule: "0 9 * * 1"                                       │
│ │   → Target: Pub/Sub topic → Cloud Function generates report     │
│ │                                                                   │
│ └── Monthly billing summary on the 1st                             │
│     → Schedule: "0 8 1 * *"                                       │
│     → Target: HTTP endpoint that sends summary email               │
│                                                                       │
│ ⚡ Cloud Scheduler vs Cloud Tasks:                                  │
│ ├── Scheduler: "Run this job every day at 2 AM" (time-based)      │
│ └── Tasks: "Process this task now, at 100/sec" (event-based)      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Creating & Managing Jobs

### Step 1: Create an HTTP Job from Console

```
Console → Navigation menu → Cloud Scheduler

┌─────────────────────────────────────────────────────────────────┐
│                CLOUD SCHEDULER — JOBS PAGE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  If first time: "Enable Cloud Scheduler API" button appears.    │
│  Click it → wait for API to enable.                              │
│                                                                   │
│  Click [+ CREATE JOB] at the top                                 │
│                                                                   │
│  ═══════════════ STEP 1: Define the schedule ═══════════════    │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Name:            [daily-report]                            │   │
│  │   └── Unique job name (lowercase, hyphens OK)             │   │
│  │                                                            │   │
│  │ Region:          [us-central1 ▼]                          │   │
│  │   └── Where the job runs (pick closest to your target)    │   │
│  │                                                            │   │
│  │ Description:     [Generate daily PDF report at 2 AM]      │   │
│  │   └── Optional but helpful for team understanding         │   │
│  │                                                            │   │
│  │ Frequency:       [0 2 * * *]                              │   │
│  │   └── Cron expression — this means "daily at 2:00 AM"    │   │
│  │                                                            │   │
│  │ Timezone:        [America/New_York ▼]                     │   │
│  │   └── IANA timezone (the cron runs in THIS timezone)      │   │
│  │   └── Use UTC for global jobs, local TZ for regional      │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Click [CONTINUE]                                                 │
│                                                                   │
│  ═══════════════ STEP 2: Configure the execution ════════════   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Target type:     ● HTTP                                    │   │
│  │                  ○ Pub/Sub                                 │   │
│  │                  ○ App Engine HTTP                         │   │
│  │                                                            │   │
│  │ URL:             [https://my-service.run.app/report]      │   │
│  │   └── The HTTP endpoint to call on each trigger           │   │
│  │                                                            │   │
│  │ HTTP method:     [POST ▼]                                 │   │
│  │   └── GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS        │   │
│  │                                                            │   │
│  │ ── Headers (optional) ──                                   │   │
│  │ Header name:     [Content-Type]                            │   │
│  │ Header value:    [application/json]                        │   │
│  │ [+ ADD HEADER]                                             │   │
│  │                                                            │   │
│  │ ── Body (optional, for POST/PUT/PATCH) ──                  │   │
│  │ Body:            [{"type":"daily","format":"pdf"}]       │   │
│  │   └── JSON, text, or any payload your endpoint expects    │   │
│  │                                                            │   │
│  │ ── Auth header (expand "Show more") ──                     │   │
│  │                                                            │   │
│  │ Auth type:       ○ No auth                                 │   │
│  │                  ● Add OIDC token ← for Cloud Run/Functions│   │
│  │                  ○ Add OAuth token ← for Google APIs       │   │
│  │                                                            │   │
│  │ Service account: [scheduler-sa@project.iam.gserviceaccount│   │
│  │                   .com ▼]                                  │   │
│  │   └── SA must have roles/run.invoker (for Cloud Run)      │   │
│  │   └── SA must have roles/cloudfunctions.invoker (for CF)  │   │
│  │                                                            │   │
│  │ Audience:        [https://my-service.run.app]             │   │
│  │   └── Usually the same as the URL (for OIDC)              │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Click [CONTINUE]                                                 │
│                                                                   │
│  ═══════════════ STEP 3: Configure retries (optional) ═══════   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Max retry attempts:    [3]                                 │   │
│  │   └── 0 = no retries, default is 0                        │   │
│  │                                                            │   │
│  │ Min backoff duration:  [5s]                                │   │
│  │   └── Wait time before first retry                        │   │
│  │                                                            │   │
│  │ Max backoff duration:  [3600s]                             │   │
│  │   └── Max wait between retries                            │   │
│  │                                                            │   │
│  │ Max doublings:         [5]                                 │   │
│  │   └── How many times backoff doubles before linear        │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Click [CREATE] → Job is created and starts ENABLED              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Create a Pub/Sub Job from Console

```
Console → Cloud Scheduler → [+ CREATE JOB]

┌─────────────────────────────────────────────────────────────────┐
│                CREATE PUB/SUB JOB                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Step 1: Define the schedule (same as HTTP job above)            │
│  ├── Name: [hourly-sync]                                        │
│  ├── Region: [us-central1]                                       │
│  ├── Frequency: [0 * * * *] (every hour)                        │
│  └── Timezone: [UTC]                                             │
│                                                                   │
│  Step 2: Configure the execution                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Target type:     ○ HTTP                                    │   │
│  │                  ● Pub/Sub  ← select this                 │   │
│  │                  ○ App Engine HTTP                         │   │
│  │                                                            │   │
│  │ Topic:           [projects/my-project/topics/sync-topic ▼]│   │
│  │   └── Select existing topic or create a new one           │   │
│  │   └── Scheduler SA needs roles/pubsub.publisher           │   │
│  │                                                            │   │
│  │ Message body:    [{"action":"sync"}]                      │   │
│  │   └── This becomes the Pub/Sub message data               │   │
│  │                                                            │   │
│  │ ── Attributes (optional) ──                                │   │
│  │ Key:   [triggered_by]   Value: [scheduler]                │   │
│  │ Key:   [type]           Value: [hourly]                   │   │
│  │ [+ ADD ATTRIBUTE]                                          │   │
│  │   └── Key-value metadata attached to the Pub/Sub message  │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Step 3: Retry config (same as HTTP job)                         │
│                                                                   │
│  Click [CREATE]                                                   │
│                                                                   │
│  💡 No auth config needed for Pub/Sub targets — Scheduler       │
│     publishes directly using its service account.                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: View Job Execution History

```
Console → Cloud Scheduler → click on job name (e.g., "daily-report")

┌─────────────────────────────────────────────────────────────────┐
│                JOB DETAILS — daily-report                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Job info:                                                        │
│  ├── State:              ENABLED (or PAUSED)                    │
│  ├── Schedule:           0 2 * * * (daily at 2 AM)              │
│  ├── Timezone:           America/New_York                        │
│  ├── Target type:        HTTP                                    │
│  ├── Target URL:         https://my-service.run.app/report      │
│  ├── Last run:           2024-01-16 02:00:00 — SUCCESS          │
│  └── Next run:           2024-01-17 02:00:00                     │
│                                                                   │
│  ══════════════════════════════════════════════════════════════  │
│                                                                   │
│  Execution history (LOGS tab):                                    │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Timestamp            │ Status  │ Response   │ Attempt    │   │
│  ├──────────────────────┼─────────┼────────────┼───────────┤   │
│  │ 2024-01-16 02:00:01  │ SUCCESS │ 200 OK     │ 1/1       │   │
│  │ 2024-01-15 02:00:01  │ SUCCESS │ 200 OK     │ 1/1       │   │
│  │ 2024-01-14 02:00:02  │ FAILED  │ 503        │ 3/3       │   │
│  │ 2024-01-13 02:00:01  │ SUCCESS │ 200 OK     │ 1/1       │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Click on a row → see detailed response:                         │
│  ├── HTTP status code                                            │
│  ├── Response body (if any)                                      │
│  ├── Retry attempts and timing                                   │
│  └── Error messages (for failures)                               │
│                                                                   │
│  💡 Also check Cloud Logging for detailed execution logs:        │
│     Logging → resource.type="cloud_scheduler_job"               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 4: Pause / Resume a Job

```
Console → Cloud Scheduler → find job → Actions menu (⋮)

┌─────────────────────────────────────────────────────────────────┐
│                PAUSE / RESUME A JOB                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PAUSE:                                                           │
│  1. Go to Cloud Scheduler page                                   │
│  2. Find the job in the list                                      │
│  3. Click the three-dot menu (⋮) on the right                    │
│  4. Select "Pause"                                                │
│                                                                   │
│  OR: Click the job name → click [PAUSE] button at the top        │
│                                                                   │
│  What happens when paused:                                        │
│  ├── Job state changes to PAUSED                                 │
│  ├── Scheduled triggers stop firing                              │
│  ├── Job is NOT deleted — it keeps its schedule and config       │
│  ├── You can still manually run a paused job                     │
│  └── Useful for: maintenance windows, debugging, cost saving     │
│                                                                   │
│  RESUME:                                                          │
│  1. Same three-dot menu (⋮) → select "Resume"                   │
│  2. OR click job name → click [RESUME] at the top                │
│  3. Job state changes back to ENABLED                             │
│  4. Next scheduled execution resumes per cron schedule            │
│                                                                   │
│  💡 Missed executions while paused are NOT retroactively run.    │
│     The job simply fires at its next scheduled time after resume. │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 5: Run a Job Manually (Test It)

```
Console → Cloud Scheduler → find job → Actions menu (⋮)

┌─────────────────────────────────────────────────────────────────┐
│                MANUALLY TRIGGER A JOB                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Go to Cloud Scheduler page                                   │
│  2. Find the job in the list                                      │
│  3. Click the three-dot menu (⋮) on the right                    │
│  4. Select "Force run"                                            │
│                                                                   │
│  OR: Click job name → click [FORCE RUN] button at the top        │
│                                                                   │
│  What happens:                                                    │
│  ├── Job fires immediately (regardless of schedule)              │
│  ├── Uses the same target, body, headers, and auth config        │
│  ├── Counts as a normal execution in the logs                    │
│  ├── Does NOT change the next scheduled run time                 │
│  └── Retries apply if the target fails                           │
│                                                                   │
│  Why use Force Run?                                               │
│  ├── Test a new job before waiting for its schedule              │
│  ├── Verify auth and target URL are correct                      │
│  ├── Re-run after a failed execution                             │
│  └── Debug target endpoint behavior                              │
│                                                                   │
│  💡 Check the "Last run" and execution history after Force Run   │
│     to confirm the result (SUCCESS or FAILED + status code).     │
│                                                                   │
│  Equivalent gcloud command:                                       │
│  gcloud scheduler jobs run daily-report --location=us-central1   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 6: Delete a Job

```
Console → Cloud Scheduler → find job → Actions menu (⋮)

┌─────────────────────────────────────────────────────────────────┐
│                DELETE A JOB                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Go to Cloud Scheduler page                                   │
│  2. Find the job in the list                                      │
│  3. Click the three-dot menu (⋮) on the right                    │
│  4. Select "Delete"                                               │
│  5. Confirm deletion in the dialog                                │
│                                                                   │
│  OR: Select multiple jobs with checkboxes → click [DELETE]       │
│                                                                   │
│  What happens:                                                    │
│  ├── Job is permanently deleted                                  │
│  ├── No more scheduled executions                                │
│  ├── Execution history is removed                                │
│  ├── Job name can be reused immediately (unlike Cloud Tasks      │
│  │   queues, there's no cooldown period)                         │
│  └── Any in-flight execution completes, but no new ones start    │
│                                                                   │
│  ⚠️  This is irreversible! No undo.                              │
│                                                                   │
│  💡 If you just want to temporarily stop a job, use PAUSE        │
│     instead of delete. You can resume a paused job anytime.      │
│                                                                   │
│  Equivalent gcloud command:                                       │
│  gcloud scheduler jobs delete daily-report --location=us-central1│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 49: Workflows** → `49-workflows.md`
