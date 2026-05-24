# Chapter 47 — Cloud Tasks

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Tasks Fundamentals](#part-1--cloud-tasks-fundamentals)
- [Part 2: Queues](#part-2--queues)
- [Part 3: Creating Tasks — HTTP Targets](#part-3--creating-tasks--http-targets)
- [Part 4: Creating Tasks — App Engine Targets](#part-4--creating-tasks--app-engine-targets)
- [Part 5: Task Scheduling & Delays](#part-5--task-scheduling--delays)
- [Part 6: Rate Limiting & Concurrency](#part-6--rate-limiting--concurrency)
- [Part 7: Retry Configuration](#part-7--retry-configuration)
- [Part 8: Task Deduplication](#part-8--task-deduplication)
- [Part 9: Authentication & Authorization](#part-9--authentication--authorization)
- [Part 10: Task Lifecycle & States](#part-10--task-lifecycle--states)
- [Part 11: Cloud Tasks vs Pub/Sub](#part-11--cloud-tasks-vs-pubsub)
- [Part 12: Cloud Tasks vs Cloud Scheduler](#part-12--cloud-tasks-vs-cloud-scheduler)
- [Part 13: Monitoring & Troubleshooting](#part-13--monitoring--troubleshooting)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What is Cloud Tasks? (Beginner Explanation)](#what-is-cloud-tasks-beginner-explanation)
- [Console Walkthrough: Creating & Managing Queues](#console-walkthrough-creating--managing-queues)
- [What's Next?](#whats-next)

---

## Overview

Cloud Tasks is a fully managed service for managing the execution, dispatch, and delivery of a large number of distributed tasks. You create tasks and add them to a queue, and Cloud Tasks reliably delivers them to your worker (HTTP endpoint or App Engine handler) with configurable rate limits, retries, and scheduling. Unlike Pub/Sub (fire-and-forget messaging), Cloud Tasks gives you explicit control over execution rate, timing, and individual task management.

---

## Part 1 — Cloud Tasks Fundamentals

### What Is Cloud Tasks?

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD TASKS OVERVIEW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Cloud Tasks = managed task queue for asynchronous work:        │
│                                                                   │
│  ┌──────────┐    ┌───────────┐    ┌──────────────────────┐     │
│  │ Producer │───►│   Queue   │───►│ Worker (HTTP target) │     │
│  │ (your    │    │           │    │                      │     │
│  │  code)   │    │ rate      │    │ Cloud Run            │     │
│  └──────────┘    │ limiting  │    │ Cloud Functions      │     │
│                  │ retry     │    │ App Engine            │     │
│                  │ scheduling│    │ Any HTTP endpoint    │     │
│                  └───────────┘    └──────────────────────┘     │
│                                                                   │
│  Key features:                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ • Rate limiting: control tasks/second dispatched         │   │
│  │ • Delayed execution: schedule tasks for the future       │   │
│  │ • Automatic retries: with exponential backoff            │   │
│  │ • Deduplication: prevent duplicate task execution        │   │
│  │ • Task management: inspect, delete individual tasks      │   │
│  │ • Authentication: OIDC tokens for secure dispatch        │   │
│  │ • At-least-once delivery guarantee                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Use when you need to:                                           │
│  → Offload work from a request path                             │
│  → Rate-limit calls to a downstream service                     │
│  → Schedule work for later execution                            │
│  → Manage and inspect individual tasks                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Cloud Tasks | AWS SQS | Azure Queue Storage |
|---------|----------------|---------|---------------------|
| Service | Cloud Tasks | SQS (+ Lambda) | Queue Storage (+ Functions) |
| Task queue | Yes | Yes | Yes |
| HTTP target | Yes (native) | No (need Lambda) | No (need Functions) |
| Rate limiting | Yes (built-in) | No (manual) | No (manual) |
| Delayed execution | Yes (up to 30 days) | Yes (up to 15 min) | Yes (up to 7 days) |
| Task deduplication | Yes (task name) | Yes (FIFO dedup ID) | No |
| Individual task mgmt | Yes (get, delete) | No (limited) | Yes (peek, delete) |
| Retry config | Yes (per queue) | Yes (redrive policy) | Yes (dequeue count) |
| Auth for dispatch | OIDC/OAuth tokens | IAM (Lambda) | Managed identity |
| App Engine target | Yes (native) | N/A | N/A |

### Pricing

| Component | Cost |
|-----------|------|
| Task operations | First 1M/month free, then $0.40/million |
| Queue management | Free |
| No charge for failed tasks | Retries are free |

---

## Part 2 — Queues

### Queue Configuration

```
┌──────────────────────────────────────────────────────────────┐
│         CLOUD TASKS QUEUES                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  A queue holds tasks and controls how they're dispatched:    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Queue: "email-queue"                                 │    │
│  │  Location: us-central1                                │    │
│  │                                                      │    │
│  │  Rate limits:                                         │    │
│  │  ├── maxDispatchesPerSecond: 500                     │    │
│  │  ├── maxBurstSize: 100                               │    │
│  │  └── maxConcurrentDispatches: 1000                   │    │
│  │                                                      │    │
│  │  Retry config:                                        │    │
│  │  ├── maxAttempts: 5                                  │    │
│  │  ├── minBackoff: 1s                                  │    │
│  │  ├── maxBackoff: 3600s                               │    │
│  │  ├── maxDoublings: 16                                │    │
│  │  └── maxRetryDuration: 86400s (1 day)                │    │
│  │                                                      │    │
│  │  State: RUNNING | PAUSED | DISABLED                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Multiple queues for different priorities / rate limits:      │
│  ├── high-priority-queue  (1000 tasks/sec)                   │
│  ├── default-queue        (100 tasks/sec)                    │
│  └── low-priority-queue   (10 tasks/sec)                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create a queue
gcloud tasks queues create email-queue \
    --location=us-central1

# Create queue with rate limits
gcloud tasks queues create batch-queue \
    --location=us-central1 \
    --max-dispatches-per-second=100 \
    --max-concurrent-dispatches=50 \
    --max-attempts=5 \
    --min-backoff=1s \
    --max-backoff=3600s \
    --max-doublings=16

# Pause a queue (tasks stay but don't dispatch)
gcloud tasks queues pause email-queue --location=us-central1

# Resume a queue
gcloud tasks queues resume email-queue --location=us-central1

# Update rate limits
gcloud tasks queues update email-queue \
    --location=us-central1 \
    --max-dispatches-per-second=500

# List queues
gcloud tasks queues list --location=us-central1

# Describe queue
gcloud tasks queues describe email-queue --location=us-central1

# Delete queue (deletes all tasks in it)
gcloud tasks queues delete email-queue --location=us-central1
```

---

## Part 3 — Creating Tasks — HTTP Targets

### HTTP Target Tasks

```
┌──────────────────────────────────────────────────────────────┐
│         HTTP TARGET TASKS                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Tasks dispatched as HTTP requests to any endpoint:           │
│                                                                │
│  ┌──────────┐         ┌─────────┐         ┌───────────────┐ │
│  │ Producer │──create──│ Queue   │──POST──►│ Cloud Run     │ │
│  │          │  task    │         │  (HTTP) │ /process-order│ │
│  └──────────┘         └─────────┘         └───────────────┘ │
│                                                                │
│  Task includes:                                                │
│  • HTTP method (GET, POST, PUT, DELETE, PATCH)               │
│  • URL (any publicly reachable HTTPS endpoint)                │
│  • Headers (optional)                                         │
│  • Body (optional, for POST/PUT/PATCH)                       │
│  • Auth token (OIDC or OAuth2)                               │
│  • Schedule time (optional, for delayed execution)           │
│                                                                │
│  Response handling:                                            │
│  • 2xx → task succeeded, removed from queue                 │
│  • 4xx (except 429) → task failed permanently, no retry     │
│  • 429, 5xx → task failed, retry per retry config           │
│  • Timeout → retry (default 10min for HTTP, 15min AppEngine)│
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create HTTP task
gcloud tasks create-http-task \
    --queue=email-queue \
    --location=us-central1 \
    --url="https://my-service.run.app/send-email" \
    --method=POST \
    --body-content='{"to":"user@example.com","subject":"Welcome"}' \
    --header="Content-Type: application/json"

# With OIDC authentication
gcloud tasks create-http-task \
    --queue=email-queue \
    --location=us-central1 \
    --url="https://my-service.run.app/send-email" \
    --method=POST \
    --body-content='{"to":"user@example.com"}' \
    --oidc-service-account-email=tasks-invoker@my-project.iam.gserviceaccount.com \
    --oidc-token-audience="https://my-service.run.app"

# With scheduled time (execute 1 hour from now)
gcloud tasks create-http-task \
    --queue=email-queue \
    --location=us-central1 \
    --url="https://my-service.run.app/send-reminder" \
    --method=POST \
    --schedule-time="2024-01-16T10:00:00Z"
```

### Python — Creating HTTP Tasks

```python
from google.cloud import tasks_v2
import json

client = tasks_v2.CloudTasksClient()
parent = client.queue_path("my-project", "us-central1", "email-queue")

task = tasks_v2.Task(
    http_request=tasks_v2.HttpRequest(
        http_method=tasks_v2.HttpMethod.POST,
        url="https://my-service.run.app/send-email",
        headers={"Content-Type": "application/json"},
        body=json.dumps({
            "to": "user@example.com",
            "subject": "Welcome"
        }).encode(),
        oidc_token=tasks_v2.OidcToken(
            service_account_email="tasks-invoker@my-project.iam.gserviceaccount.com",
            audience="https://my-service.run.app",
        ),
    ),
)

response = client.create_task(parent=parent, task=task)
print(f"Created task: {response.name}")
```

---

## Part 4 — Creating Tasks — App Engine Targets

### App Engine Target Tasks

```bash
# Create App Engine task
gcloud tasks create-app-engine-task \
    --queue=default \
    --location=us-central1 \
    --relative-uri=/worker/process \
    --method=POST \
    --body-content='{"jobId":"12345"}' \
    --header="Content-Type: application/json" \
    --routing="service:worker,version:v2"

# App Engine routing headers
# service: target App Engine service
# version: target version
# instance: target instance (rare)
```

```
App Engine tasks are dispatched internally — no public URL needed.
Cloud Tasks sends the request directly to the App Engine service
using internal routing. No OIDC token needed.
```

---

## Part 5 — Task Scheduling & Delays

### Delayed Execution

```
┌──────────────────────────────────────────────────────────────┐
│         TASK SCHEDULING                                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Schedule tasks to execute at a specific future time:        │
│                                                                │
│  Now ──────────────────────────────────────── Future          │
│   │                                           │               │
│   │ create task                               │ task executes │
│   │ schedule_time = now + 1 hour              │               │
│   ▼                                           ▼               │
│  [Task created]     [Held in queue]      [Dispatched]        │
│                                                                │
│  Max delay: 30 days from creation                             │
│                                                                │
│  Use cases:                                                    │
│  • Send reminder email 24 hours after signup                 │
│  • Retry a payment in 1 hour                                 │
│  • Schedule report generation for midnight                   │
│  • Delayed cleanup of temporary resources                    │
│  • Timeout handling (create task to check status after X)    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```python
from google.cloud import tasks_v2
from google.protobuf import timestamp_pb2
import datetime

client = tasks_v2.CloudTasksClient()
parent = client.queue_path("my-project", "us-central1", "reminders")

# Schedule 24 hours from now
schedule_time = timestamp_pb2.Timestamp()
schedule_time.FromDatetime(
    datetime.datetime.utcnow() + datetime.timedelta(hours=24)
)

task = tasks_v2.Task(
    http_request=tasks_v2.HttpRequest(
        http_method=tasks_v2.HttpMethod.POST,
        url="https://my-service.run.app/send-reminder",
        body=b'{"userId": "user123"}',
    ),
    schedule_time=schedule_time,
)

response = client.create_task(parent=parent, task=task)
```

---

## Part 6 — Rate Limiting & Concurrency

### Rate Control

```
┌──────────────────────────────────────────────────────────────┐
│         RATE LIMITING                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Three rate controls on each queue:                           │
│                                                                │
│  1. maxDispatchesPerSecond                                    │
│     → Max tasks dispatched per second                        │
│     → Default: 500, Max: 500                                 │
│     → Set lower to protect downstream services               │
│                                                                │
│  2. maxBurstSize                                              │
│     → Token bucket burst allowance                           │
│     → Allows short bursts above the rate limit               │
│     → Default: auto-calculated                               │
│                                                                │
│  3. maxConcurrentDispatches                                   │
│     → Max tasks executing simultaneously                     │
│     → Default: 1000, Max: 5000                               │
│     → Controls how many inflight requests to worker          │
│                                                                │
│  Example: Protecting an external API                          │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  External API limit: 100 requests/second              │    │
│  │                                                      │    │
│  │  Queue config:                                        │    │
│  │  maxDispatchesPerSecond: 80   (leave 20% headroom)   │    │
│  │  maxConcurrentDispatches: 50  (limit parallel calls) │    │
│  │                                                      │    │
│  │  → Cloud Tasks smooths out traffic to the API        │    │
│  │  → No risk of hitting API rate limits                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 7 — Retry Configuration

### Retry Policy

```
┌──────────────────────────────────────────────────────────────┐
│         RETRY CONFIGURATION                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  When a task fails (5xx, timeout), Cloud Tasks retries:      │
│                                                                │
│  Attempt 1: immediate ──► FAIL                               │
│  Attempt 2: wait 1s   ──► FAIL  (minBackoff)                │
│  Attempt 3: wait 2s   ──► FAIL  (doubled)                   │
│  Attempt 4: wait 4s   ──► FAIL  (doubled)                   │
│  Attempt 5: wait 8s   ──► FAIL  (max attempts reached)      │
│  → Task permanently fails, removed from queue               │
│                                                                │
│  Parameters:                                                   │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ maxAttempts        │ Max retries (-1 = unlimited)    │    │
│  │ minBackoff         │ Min wait between retries (0.1s) │    │
│  │ maxBackoff         │ Max wait between retries (1h)   │    │
│  │ maxDoublings       │ Times backoff doubles (16)      │    │
│  │ maxRetryDuration   │ Max total time retrying         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Response codes and retry behavior:                           │
│  • 2xx  → Success, task complete                             │
│  • 429  → Rate limited, retry with backoff                   │
│  • 5xx  → Server error, retry with backoff                   │
│  • 4xx  → Client error, NO retry (permanent failure)        │
│    (except 429)                                               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 8 — Task Deduplication

### Using Task Names for Deduplication

```bash
# Create task with explicit name (deduplication)
gcloud tasks create-http-task \
    --queue=orders-queue \
    --location=us-central1 \
    --url="https://my-service.run.app/process" \
    --method=POST \
    --task-id="order-12345" \
    --body-content='{"orderId":"12345"}'

# If you create another task with the same name while the
# original is still in the queue → ALREADY_EXISTS error
# This prevents duplicate task creation

# After a task completes, the name is tombstoned for ~1 hour
# During tombstone period: re-creating the same name → error
# After tombstone expires: you can reuse the name
```

```
Deduplication rules:
• Task name must be unique within the queue at any given time
• After task completes/fails: name tombstoned for ~1 hour
• Use deterministic naming: "process-order-{orderId}"
• Callers should handle ALREADY_EXISTS gracefully
```

---

## Part 9 — Authentication & Authorization

### Securing Task Dispatch

```
┌──────────────────────────────────────────────────────────────┐
│         AUTHENTICATION                                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  OIDC Token (recommended for Cloud Run / Cloud Functions):   │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Cloud Tasks ──OIDC token──► Cloud Run               │    │
│  │                                                      │    │
│  │  Task includes:                                      │    │
│  │  • service_account_email (who to impersonate)        │    │
│  │  • audience (target service URL)                     │    │
│  │                                                      │    │
│  │  Cloud Run verifies the OIDC token automatically     │    │
│  │  The SA needs roles/run.invoker on the service       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  OAuth Token (for Google APIs):                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Cloud Tasks ──OAuth2 token──► Google API             │    │
│  │  (e.g., calling Sheets API, Gmail API)               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Required IAM:                                                 │
│  • Task creator needs roles/cloudtasks.enqueuer on queue    │
│  • SA in task needs roles/iam.serviceAccountUser             │
│  • SA needs invoke permission on target service              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 10 — Task Lifecycle & States

### Task States

```
┌──────────────────────────────────────────────────────────────┐
│         TASK LIFECYCLE                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Created ──► Scheduled ──► Dispatched ──► Running            │
│                                              │                │
│                                     ┌────────┴────────┐      │
│                                     ▼                 ▼      │
│                                  SUCCESS           FAILURE   │
│                                  (2xx)             (5xx/429)│
│                                     │                 │      │
│                                     ▼                 ▼      │
│                                  Completed        Retry?     │
│                                  (removed)      ┌────┴────┐  │
│                                                 ▼         ▼  │
│                                               Yes        No  │
│                                           (re-queue) (removed)│
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Inspect a specific task
gcloud tasks describe TASK_ID \
    --queue=email-queue \
    --location=us-central1

# List tasks in a queue
gcloud tasks list \
    --queue=email-queue \
    --location=us-central1

# Delete a specific task
gcloud tasks delete TASK_ID \
    --queue=email-queue \
    --location=us-central1

# Purge all tasks from a queue
gcloud tasks queues purge email-queue \
    --location=us-central1
```

---

## Part 11 — Cloud Tasks vs Pub/Sub

### When to Use Which

```
┌──────────────────────────────────────────────────────────────┐
│         CLOUD TASKS vs PUB/SUB                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────┬──────────────────────────────┐     │
│  │ Cloud Tasks          │ Pub/Sub                       │     │
│  ├──────────────────────┼──────────────────────────────┤     │
│  │ Explicit rate control│ No built-in rate control     │     │
│  │ Scheduled delivery   │ Immediate delivery           │     │
│  │ Task management      │ No individual msg management │     │
│  │ One consumer/task    │ Fan-out to many consumers    │     │
│  │ HTTP dispatch        │ Pull or push                  │     │
│  │ Deduplication        │ No built-in dedup            │     │
│  │ At-least-once        │ At-least-once or exactly-once│     │
│  ├──────────────────────┼──────────────────────────────┤     │
│  │ USE FOR:             │ USE FOR:                      │     │
│  │ • Rate-limited APIs  │ • Event streaming            │     │
│  │ • Scheduled jobs     │ • Fan-out broadcasting       │     │
│  │ • Task management    │ • Decoupled microservices    │     │
│  │ • Deferred work      │ • Log aggregation            │     │
│  │ • Webhook delivery   │ • Data pipelines             │     │
│  └──────────────────────┴──────────────────────────────┘     │
│                                                                │
│  Rule of thumb:                                                │
│  → Need to control WHEN and HOW FAST? → Cloud Tasks         │
│  → Need to broadcast to MANY consumers? → Pub/Sub           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 12 — Cloud Tasks vs Cloud Scheduler

```
┌──────────────────────────────────────────────────────────────┐
│         CLOUD TASKS vs CLOUD SCHEDULER                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌───────────────────────┬─────────────────────────────┐     │
│  │ Cloud Tasks           │ Cloud Scheduler              │     │
│  ├───────────────────────┼─────────────────────────────┤     │
│  │ Programmatic creation │ Cron-based schedule          │     │
│  │ Created by your code  │ Created manually/Terraform   │     │
│  │ One-shot execution    │ Recurring execution          │     │
│  │ Millions of tasks     │ Handful of scheduled jobs    │     │
│  │ Variable timing       │ Fixed schedule (cron syntax) │     │
│  ├───────────────────────┼─────────────────────────────┤     │
│  │ USE FOR:              │ USE FOR:                     │     │
│  │ • Process each order  │ • Nightly DB backup          │     │
│  │ • Send each email     │ • Hourly data sync           │     │
│  │ • Retry each webhook  │ • Daily report generation    │     │
│  │ • Deferred cleanup    │ • Weekly cleanup jobs        │     │
│  └───────────────────────┴─────────────────────────────┘     │
│                                                                │
│  They complement each other:                                   │
│  Cloud Scheduler → triggers job → creates Cloud Tasks         │
│  (e.g., every hour, create tasks for each pending order)     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Monitoring & Troubleshooting

### Key Metrics

| Metric | Description |
|--------|-------------|
| `cloudtasks.googleapis.com/queue/depth` | Tasks waiting in queue |
| `cloudtasks.googleapis.com/queue/task_attempt_count` | Attempts per task |
| `cloudtasks.googleapis.com/api/request_count` | API requests to Cloud Tasks |
| `cloudtasks.googleapis.com/queue/task_attempt_delays` | Time tasks wait before dispatch |

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Tasks not dispatching | Queue paused | `gcloud tasks queues resume QUEUE` |
| Tasks accumulating | Worker too slow | Increase `maxConcurrentDispatches` |
| PERMISSION_DENIED on dispatch | SA missing invoke role | Grant `roles/run.invoker` to task SA |
| DEADLINE_EXCEEDED | Worker takes too long | Increase task `dispatch_deadline` |
| ALREADY_EXISTS | Duplicate task name | Task still in queue or tombstoned |

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Complete Setup

```hcl
# ─── Queue ────────────────────────────────────────────────────
resource "google_cloud_tasks_queue" "email" {
  name     = "email-queue"
  location = var.region
  project  = var.project_id

  rate_limits {
    max_dispatches_per_second = 100
    max_concurrent_dispatches = 50
    max_burst_size            = 20
  }

  retry_config {
    max_attempts       = 5
    min_backoff        = "1s"
    max_backoff        = "3600s"
    max_doublings      = 16
    max_retry_duration = "86400s"
  }

  stackdriver_logging_config {
    sampling_ratio = 1.0  # Log all task attempts
  }
}

# ─── Service Account for Task Dispatch ────────────────────────
resource "google_service_account" "tasks_invoker" {
  account_id   = "tasks-invoker"
  display_name = "Cloud Tasks Invoker"
  project      = var.project_id
}

# Grant SA permission to invoke Cloud Run
resource "google_cloud_run_service_iam_member" "invoker" {
  project  = var.project_id
  location = var.region
  service  = google_cloud_run_v2_service.worker.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.tasks_invoker.email}"
}

# Grant task creator permission to use the SA
resource "google_service_account_iam_member" "task_creator" {
  service_account_id = google_service_account.tasks_invoker.name
  role               = "roles/iam.serviceAccountUser"
  member             = "serviceAccount:${google_service_account.api_service.email}"
}

# ─── IAM — Queue Access ──────────────────────────────────────
resource "google_cloud_tasks_queue_iam_member" "enqueuer" {
  project  = var.project_id
  location = var.region
  name     = google_cloud_tasks_queue.email.name
  role     = "roles/cloudtasks.enqueuer"
  member   = "serviceAccount:${google_service_account.api_service.email}"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# QUEUES
# ═══════════════════════════════════════════════════════════════
gcloud tasks queues create QUEUE --location=LOC [options]
gcloud tasks queues update QUEUE --location=LOC [options]
gcloud tasks queues describe QUEUE --location=LOC
gcloud tasks queues list --location=LOC
gcloud tasks queues delete QUEUE --location=LOC
gcloud tasks queues pause QUEUE --location=LOC
gcloud tasks queues resume QUEUE --location=LOC
gcloud tasks queues purge QUEUE --location=LOC

# ═══════════════════════════════════════════════════════════════
# HTTP TASKS
# ═══════════════════════════════════════════════════════════════
gcloud tasks create-http-task \
    --queue=Q --location=LOC \
    --url=URL --method=POST \
    --body-content=BODY \
    --header="Content-Type: application/json" \
    --oidc-service-account-email=SA \
    --schedule-time=TIME \
    --task-id=ID

# ═══════════════════════════════════════════════════════════════
# APP ENGINE TASKS
# ═══════════════════════════════════════════════════════════════
gcloud tasks create-app-engine-task \
    --queue=Q --location=LOC \
    --relative-uri=/path \
    --method=POST --body-content=BODY

# ═══════════════════════════════════════════════════════════════
# TASK MANAGEMENT
# ═══════════════════════════════════════════════════════════════
gcloud tasks list --queue=Q --location=LOC
gcloud tasks describe TASK --queue=Q --location=LOC
gcloud tasks delete TASK --queue=Q --location=LOC
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Email Queue with Rate Limiting

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: RATE-LIMITED EMAIL SENDING                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Email provider (SendGrid) rate limit: 100 emails/second             │
│                                                                        │
│  ┌──────────┐    ┌──────────────────────┐    ┌────────────┐         │
│  │ Order API│    │ email-queue           │    │ Email      │         │
│  │          │───►│ maxDispatches: 80/sec │───►│ Worker     │         │
│  └──────────┘    │ maxConcurrent: 40     │    │ (Cloud Run)│         │
│                  │ retries: 3            │    │            │         │
│  ┌──────────┐───►│                       │    │ Calls      │         │
│  │ Signup   │    └──────────────────────┘    │ SendGrid   │         │
│  │ Service  │                                 │ API        │         │
│  └──────────┘                                 └────────────┘         │
│                                                                        │
│  Benefits:                                                            │
│  • Never exceed SendGrid rate limit                                  │
│  • Spikes in signups don't overwhelm email service                   │
│  • Automatic retry on transient failures                             │
│  • Task dedup prevents duplicate emails (task-id = "welcome-{uid}") │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Webhook Delivery System

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: RELIABLE WEBHOOK DELIVERY                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  SaaS platform delivers webhooks to customer endpoints:              │
│                                                                        │
│  Event occurs → create task per customer webhook URL                 │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Queue: "webhooks"                                        │        │
│  │  maxDispatchesPerSecond: 200                              │        │
│  │  maxConcurrentDispatches: 100                             │        │
│  │  maxAttempts: 8                                           │        │
│  │  minBackoff: 10s                                          │        │
│  │  maxBackoff: 3600s (1 hour)                               │        │
│  │  maxRetryDuration: 86400s (24 hours)                      │        │
│  │                                                           │        │
│  │  Retry schedule:                                          │        │
│  │  10s → 20s → 40s → 80s → 160s → 320s → 640s → 1280s    │        │
│  │                                                           │        │
│  │  Task name: "webhook-{eventId}-{customerId}"              │        │
│  │  → Prevents duplicate webhooks for same event            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  After max retries: log failed delivery, notify customer             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Deferred Work Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: DEFERRED WORK FROM API REQUESTS                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  API receives request → responds immediately → defers heavy work    │
│                                                                        │
│  User uploads file:                                                   │
│  ┌───────────┐  POST /upload  ┌───────────┐                         │
│  │ Client    │──────────────►│ API       │                           │
│  │           │◄──────────────│ (Cloud Run│  Response: 202 Accepted  │
│  └───────────┘  {"status":   │  200ms)   │                           │
│                 "processing"} └─────┬─────┘                           │
│                                     │ create tasks                   │
│                           ┌─────────┼─────────┐                     │
│                           ▼         ▼         ▼                     │
│                    ┌──────────┐┌──────────┐┌──────────┐             │
│                    │ Virus    ││ Thumbnail││ Index in │             │
│                    │ Scan     ││ Generate ││ Search   │             │
│                    │ Task     ││ Task     ││ Task     │             │
│                    │ (now)    ││ (now)    ││ (+5 min) │             │
│                    └──────────┘└──────────┘└──────────┘             │
│                                                                        │
│  Benefits:                                                            │
│  • API responds fast (sub-second)                                    │
│  • Heavy work happens asynchronously                                 │
│  • Tasks can be scheduled with delays                                │
│  • Each task retries independently                                   │
│  • Rate limiting protects downstream services                        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create queue | `gcloud tasks queues create Q --location=L` |
| Create HTTP task | `gcloud tasks create-http-task --queue=Q --url=U --method=POST` |
| Schedule task | `--schedule-time="2024-01-16T10:00:00Z"` |
| Deduplicate | `--task-id="unique-name"` |
| Auth (OIDC) | `--oidc-service-account-email=SA --oidc-token-audience=URL` |
| Rate limit | `--max-dispatches-per-second=100` |
| Concurrency | `--max-concurrent-dispatches=50` |
| Retry | `--max-attempts=5 --min-backoff=1s --max-backoff=3600s` |
| Pause queue | `gcloud tasks queues pause Q --location=L` |
| Purge queue | `gcloud tasks queues purge Q --location=L` |
| List tasks | `gcloud tasks list --queue=Q --location=L` |
| Max delay | 30 days |
| Free tier | 1M tasks/month |
| Pricing | $0.40 per million tasks |

---

## What is Cloud Tasks? (Beginner Explanation)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD TASKS — THE SIMPLE EXPLANATION                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 💡 Cloud Tasks is like a to-do list for your servers — you add     │
│    tasks, and your server processes them when it's ready.           │
│                                                                       │
│ Think of a restaurant:                                              │
│ ├── Customer places an order (= producer creates a task)           │
│ ├── Order goes on the ticket rail (= task enters the queue)        │
│ ├── Kitchen works through orders one by one (= worker processes)   │
│ └── If a dish is wrong, kitchen remakes it (= automatic retry)     │
│                                                                       │
│ Why does this matter?                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ 1. RATE LIMITING — Don't overwhelm your servers               │   │
│ │    Without Cloud Tasks:                                       │   │
│ │    → 10,000 users click "send email" at the same time        │   │
│ │    → Your email service crashes under the load               │   │
│ │                                                               │   │
│ │    With Cloud Tasks:                                          │   │
│ │    → 10,000 tasks enter the queue                             │   │
│ │    → Queue dispatches 100/second (your configured rate)      │   │
│ │    → Email service handles them smoothly                     │   │
│ │                                                               │   │
│ │ 2. AUTOMATIC RETRIES — Handle failures gracefully             │   │
│ │    Without Cloud Tasks:                                       │   │
│ │    → API call fails → you lose the request                   │   │
│ │    → You have to build retry logic yourself                  │   │
│ │                                                               │   │
│ │    With Cloud Tasks:                                          │   │
│ │    → Task fails → Cloud Tasks retries automatically          │   │
│ │    → Exponential backoff (wait 1s, 2s, 4s, 8s...)           │   │
│ │    → Configurable max attempts                                │   │
│ │                                                               │   │
│ │ 3. DEFERRED EXECUTION — Schedule work for later               │   │
│ │    → User signs up now → send welcome email in 1 hour        │   │
│ │    → Payment received → generate invoice tomorrow at 9 AM   │   │
│ │    → Schedule tasks up to 30 days in the future              │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Real-world example:                                                │
│                                                                       │
│   User clicks               Cloud Tasks               Your worker  │
│   "Place Order"             queue                     (Cloud Run)  │
│       │                        │                          │         │
│       ▼                        ▼                          ▼         │
│   ┌────────┐  add task   ┌──────────┐  dispatch     ┌──────────┐  │
│   │  API   │────────────▶│  Queue   │──────────────▶│ Process  │  │
│   │ (fast) │ respond 200 │ rate:100 │  at safe rate │  Order   │  │
│   └────────┘  instantly  │ retry:5  │               │ Send email│  │
│                          └──────────┘               │ Update DB │  │
│                                                      └──────────┘  │
│                                                                       │
│   The user gets an instant response ("Order received!")            │
│   The heavy work happens in the background, at a safe pace.        │
│                                                                       │
│ ⚡ Cloud Tasks vs Pub/Sub — When to use which?                     │
│ ├── Cloud Tasks: you control the rate, timing, and retries         │
│ │   → "Process this task at 100/sec with 5 retries"               │
│ └── Pub/Sub: fire-and-forget messaging between services            │
│     → "Broadcast this event to all subscribers"                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Creating & Managing Queues

### Step 1: Create a Queue from Console

```
Console → Navigation menu → Cloud Tasks

┌─────────────────────────────────────────────────────────────────┐
│                CLOUD TASKS — QUEUES PAGE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  If first time: "Enable Cloud Tasks API" button appears.        │
│  Click it → wait for API to enable.                              │
│                                                                   │
│  Once enabled:                                                    │
│  Click [+ CREATE QUEUE] at the top                               │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Queue ID:        [email-queue]                             │   │
│  │   └── Unique name for this queue (lowercase, hyphens OK)  │   │
│  │                                                            │   │
│  │ Region:          [us-central1 ▼]                          │   │
│  │   └── Where the queue lives (pick closest to your worker) │   │
│  │                                                            │   │
│  │ ══════════════════════════════════════════════════════════ │   │
│  │ Rate limits (optional — expand "Rate limits"):             │   │
│  │                                                            │   │
│  │ Max dispatches per second:   [500]                        │   │
│  │   └── How many tasks/sec the queue sends to your worker   │   │
│  │                                                            │   │
│  │ Max concurrent dispatches:   [1000]                       │   │
│  │   └── How many tasks can be in-flight at the same time    │   │
│  │                                                            │   │
│  │ Max burst size:              [100]                        │   │
│  │   └── Burst capacity (auto-calculated if left empty)      │   │
│  │                                                            │   │
│  │ ══════════════════════════════════════════════════════════ │   │
│  │ Retry config (optional — expand "Retry configuration"):   │   │
│  │                                                            │   │
│  │ Max attempts:                [5]                          │   │
│  │   └── How many times to retry a failed task (-1=unlimited)│   │
│  │                                                            │   │
│  │ Min backoff:                 [1s]                         │   │
│  │   └── Wait time before first retry                        │   │
│  │                                                            │   │
│  │ Max backoff:                 [3600s]                      │   │
│  │   └── Maximum wait between retries                        │   │
│  │                                                            │   │
│  │ Max doublings:               [16]                         │   │
│  │   └── How many times backoff doubles before going linear  │   │
│  │                                                            │   │
│  │ Max retry duration:          [86400s]                     │   │
│  │   └── Stop retrying after this total time (0=no limit)    │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Click [CREATE] → Queue is created and starts in RUNNING state   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: View Queue Details and Tasks

```
Console → Cloud Tasks → click on queue name (e.g., "email-queue")

┌─────────────────────────────────────────────────────────────────┐
│                QUEUE DETAILS — email-queue                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Queue info:                                                      │
│  ├── State:              RUNNING (or PAUSED)                    │
│  ├── Region:             us-central1                             │
│  ├── Tasks in queue:     142                                     │
│  ├── Max rate:           500/sec                                 │
│  └── Max concurrent:     1000                                    │
│                                                                   │
│  ══════════════════════════════════════════════════════════════  │
│                                                                   │
│  TASKS tab — shows all pending tasks:                            │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Task Name        │ Schedule Time    │ Create Time         │   │
│  ├──────────────────┼─────────────────┼────────────────────┤   │
│  │ task-001         │ 2024-01-16 10:00│ 2024-01-16 09:30   │   │
│  │ task-002         │ (immediate)     │ 2024-01-16 09:31   │   │
│  │ task-003         │ 2024-01-17 08:00│ 2024-01-16 09:32   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Click a task → see full details:                                │
│  ├── HTTP method & URL                                          │
│  ├── Headers & body                                              │
│  ├── Dispatch count (how many attempts)                         │
│  ├── Response status of last attempt                             │
│  └── Schedule time & create time                                 │
│                                                                   │
│  CONFIGURATION tab — shows queue settings:                       │
│  ├── Rate limits (dispatches/sec, concurrent, burst)            │
│  └── Retry config (max attempts, backoff, doublings)            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: Pause / Resume a Queue

```
Console → Cloud Tasks → select queue → click [PAUSE QUEUE]

┌─────────────────────────────────────────────────────────────────┐
│                PAUSE / RESUME A QUEUE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PAUSE:                                                           │
│  1. Go to Cloud Tasks page                                       │
│  2. Click on the queue name (e.g., "email-queue")               │
│  3. Click [PAUSE QUEUE] button at the top                        │
│  4. Confirm the action                                            │
│                                                                   │
│  What happens when paused:                                        │
│  ├── Queue state changes to PAUSED                               │
│  ├── No new tasks are dispatched to workers                     │
│  ├── Existing tasks remain in the queue (not lost!)             │
│  ├── You can still ADD new tasks to a paused queue              │
│  └── Useful for: debugging, worker maintenance, overload        │
│                                                                   │
│  RESUME:                                                          │
│  1. Click [RESUME QUEUE] button (replaces Pause button)         │
│  2. Queue state changes back to RUNNING                          │
│  3. Tasks start dispatching again immediately                    │
│                                                                   │
│  💡 Pause/Resume does NOT delete any tasks.                      │
│     Think of it as pressing "pause" on a music player.          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 4: Purge a Queue (Delete All Tasks)

```
Console → Cloud Tasks → select queue → click [PURGE QUEUE]

┌─────────────────────────────────────────────────────────────────┐
│                PURGE A QUEUE                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Go to Cloud Tasks page                                       │
│  2. Click on the queue name                                      │
│  3. Click [PURGE QUEUE] button at the top                        │
│  4. Confirm: "Are you sure you want to purge all tasks?"        │
│                                                                   │
│  What happens:                                                    │
│  ├── ALL tasks in the queue are permanently deleted             │
│  ├── The queue itself remains (it's not deleted)                │
│  ├── Queue configuration (rate limits, retry) unchanged         │
│  ├── Queue state remains RUNNING                                 │
│  └── New tasks can be added immediately after purging           │
│                                                                   │
│  ⚠️  This is irreversible! All pending tasks are lost.          │
│                                                                   │
│  When to use:                                                     │
│  ├── Bad deployment pushed thousands of broken tasks            │
│  ├── Testing environment cleanup                                 │
│  ├── Duplicate tasks accidentally created                        │
│  └── Need to start fresh without deleting queue config          │
│                                                                   │
│  Purge vs Pause:                                                  │
│  ├── Pause = stop dispatching, keep tasks (temporary)           │
│  └── Purge = delete all tasks, keep queue (permanent)           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 5: Delete a Queue

```
Console → Cloud Tasks → select queue → click [DELETE]

┌─────────────────────────────────────────────────────────────────┐
│                DELETE A QUEUE                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Go to Cloud Tasks page                                       │
│  2. Check the box next to the queue name                         │
│  3. Click [DELETE] button at the top (trash icon)                │
│  4. Type the queue name to confirm                                │
│  5. Click [DELETE]                                                │
│                                                                   │
│  What happens:                                                    │
│  ├── Queue and ALL its tasks are permanently deleted            │
│  ├── Queue name cannot be reused for ~7 days                    │
│  │   (GCP enforces a cooldown period on deleted queue names)    │
│  └── Any code still adding tasks to this queue will get errors  │
│                                                                   │
│  ⚠️  This is irreversible!                                       │
│                                                                   │
│  💡 If you just want to clear tasks, use PURGE instead.         │
│     Delete only when you no longer need the queue at all.        │
│                                                                   │
│  Important: The 7-day name cooldown means if you delete          │
│  "email-queue", you can't create a new queue called              │
│  "email-queue" for about a week. Plan accordingly!               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 48: Cloud Scheduler** → `48-cloud-scheduler.md`
