# Chapter 39 — Error Reporting & Profiler

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Error Reporting Fundamentals](#part-1--error-reporting-fundamentals)
- [Part 2: Error Grouping & Error Detail](#part-2--error-grouping--error-detail)
- [Part 3: Error Notifications](#part-3--error-notifications)
- [Part 4: Reporting Errors from Applications](#part-4--reporting-errors-from-applications)
- [Part 5: Error Reporting for GCP Services](#part-5--error-reporting-for-gcp-services)
- [Part 6: Error Reporting API](#part-6--error-reporting-api)
- [Part 7: Resolution & Muting](#part-7--resolution--muting)
- [Part 8: Cloud Profiler Fundamentals](#part-8--cloud-profiler-fundamentals)
- [Part 9: Profiler Architecture & Profiling Types](#part-9--profiler-architecture--profiling-types)
- [Part 10: Instrumenting Applications for Profiler](#part-10--instrumenting-applications-for-profiler)
- [Part 11: Profiler Console & Analysis](#part-11--profiler-console--analysis)
- [Part 12: Comparing Profiles & Flame Graphs](#part-12--comparing-profiles--flame-graphs)
- [Part 13: IAM & Security](#part-13--iam--security)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

This chapter covers two complementary observability services:

**Cloud Error Reporting** automatically groups and counts application errors, tracks new and recurring issues, and sends notifications. It parses exception stack traces from Cloud Logging entries and presents a prioritized dashboard of errors.

**Cloud Profiler** is a continuous, statistical, low-overhead profiling service that analyses CPU, memory (heap), and wall-clock time usage across production workloads, helping you find hot functions and optimise performance without impacting your application.

---

## Part 1 — Error Reporting Fundamentals

### What Is Cloud Error Reporting?

Cloud Error Reporting scans log entries for exceptions and stack traces, groups them by root cause, and tracks their frequency over time:

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD ERROR REPORTING OVERVIEW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐    │
│  │ Cloud Run  │  │ GKE Pods   │  │ Compute Engine VMs     │    │
│  │ App Engine │  │            │  │ Cloud Functions        │    │
│  └─────┬──────┘  └─────┬──────┘  └──────────┬─────────────┘    │
│        │                │                     │                  │
│        └────────────────┼─────────────────────┘                  │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Cloud Logging                           │    │
│  │  (log entries with severity ERROR + stack traces)       │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Error Reporting                             │    │
│  │                                                         │    │
│  │  1. Parse stack traces from log entries                 │    │
│  │  2. Group errors by root cause (same exception          │    │
│  │     + stack trace location)                             │    │
│  │  3. Count occurrences, track first/last seen            │    │
│  │  4. Detect NEW errors (never seen before)               │    │
│  │  5. Send notifications (email, mobile, Pub/Sub)         │    │
│  │                                                         │    │
│  │  ┌───────────────────────────────────────────────┐     │    │
│  │  │  Error Dashboard                               │     │    │
│  │  │  ────────────────────────────────────────────  │     │    │
│  │  │  ⚠ NullPointerException (app.py:42)    1,234  │     │    │
│  │  │  ⚠ ConnectionRefused (db.py:87)          456  │     │    │
│  │  │  🆕 TimeoutError (payment.py:15)           12  │     │    │
│  │  │  ✓ IndexError (utils.py:33)  [resolved]    0  │     │    │
│  │  └───────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Capabilities

| Capability | Description |
|------------|-------------|
| **Automatic grouping** | Groups identical errors by exception type + stack trace |
| **New error detection** | Flags errors that have never been seen before |
| **Occurrence counting** | Tracks error frequency over time |
| **Stack trace parsing** | Supports Python, Java, Go, Node.js, Ruby, PHP, C# |
| **Notifications** | Email, mobile push, Pub/Sub for new/recurring errors |
| **Resolution tracking** | Mark errors as resolved; re-opens if they recur |
| **Muting** | Suppress notifications for known/expected errors |
| **Linking** | Links to associated log entries and source code |
| **Filtering** | Filter by service, version, time range |

### Cross-Cloud Comparison

| Feature | GCP Error Reporting | AWS (No Direct Equivalent) | Azure Application Insights |
|---------|--------------------|-----------------------------|---------------------------|
| Service | Error Reporting | CloudWatch Logs + custom parsing | App Insights Failures blade |
| Grouping | Automatic (stack trace) | Manual / Lambda Insights | Automatic (AI-based) |
| Stack trace parsing | Built-in (7+ languages) | Custom implementation | Built-in |
| New error detection | Yes (with notification) | Custom CloudWatch alarm | Smart detection |
| Resolution tracking | Yes | No | Yes (via work items) |
| Integration | Cloud Logging | CloudWatch Logs | Log Analytics |
| Pricing | Free | N/A | Part of App Insights pricing |

### Pricing

| Component | Cost |
|-----------|------|
| Error Reporting | **Free** |
| No ingestion charge | Reads from Cloud Logging (already ingested) |
| Notifications | Free |
| API calls | Free |

> **Note**: Error Reporting itself is completely free. You only pay for the underlying Cloud Logging ingestion of the log entries that contain the errors.

---

## Part 2 — Error Grouping & Error Detail

### How Error Grouping Works

```
┌──────────────────────────────────────────────────────────────┐
│              ERROR GROUPING LOGIC                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Error Reporting groups errors by:                              │
│  1. Exception type (class name)                                │
│  2. Stack trace top frames (where the error originated)       │
│  3. Service name + version                                    │
│                                                                │
│  Example — these two are grouped together:                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Traceback (most recent call last):                   │    │
│  │    File "app.py", line 42, in process_order          │    │
│  │      order = db.get_order(order_id)                  │    │
│  │    File "db.py", line 87, in get_order               │    │
│  │      cursor.execute(query)                           │    │
│  │  psycopg2.OperationalError: connection refused       │    │
│  └──────────────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Traceback (most recent call last):                   │    │
│  │    File "app.py", line 42, in process_order          │    │
│  │      order = db.get_order(order_id)                  │    │
│  │    File "db.py", line 87, in get_order               │    │
│  │      cursor.execute(query)                           │    │
│  │  psycopg2.OperationalError: connection refused       │    │
│  └──────────────────────────────────────────────────────┘    │
│  → Same exception + same stack location = 1 error group      │
│                                                                │
│  This is a DIFFERENT group (different stack location):        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Traceback (most recent call last):                   │    │
│  │    File "app.py", line 105, in update_inventory      │    │
│  │      db.update(item_id, count)                       │    │
│  │    File "db.py", line 87, in update                  │    │
│  │      cursor.execute(query)                           │    │
│  │  psycopg2.OperationalError: connection refused       │    │
│  └──────────────────────────────────────────────────────┘    │
│  → Same exception but different call site = separate group   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Error Detail View

```
┌──────────────────────────────────────────────────────────────┐
│              ERROR DETAIL VIEW                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Console → Error Reporting → click an error group             │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  psycopg2.OperationalError: connection refused       │    │
│  │                                                      │    │
│  │  Status: [Open ▼]   Muted: No                       │    │
│  │  First seen: 2026-05-15 08:23:00                     │    │
│  │  Last seen:  2026-05-17 10:45:12                     │    │
│  │  Occurrences: 1,234 (last 24h: 89)                   │    │
│  │  Affected users: N/A                                  │    │
│  │  Services: checkout-api (v2.1.0)                     │    │
│  │                                                      │    │
│  │  Occurrence timeline:                                │    │
│  │  ▂▃▅▇█▆▃▂▁▂▃▅▇█▅▃▂▁                                │    │
│  │  May 15    May 16    May 17                          │    │
│  │                                                      │    │
│  │  Stack trace:                                        │    │
│  │  ┌────────────────────────────────────────────────┐ │    │
│  │  │  File "app.py", line 42, in process_order      │ │    │
│  │  │    order = db.get_order(order_id)  [View source]│ │    │
│  │  │  File "db.py", line 87, in get_order           │ │    │
│  │  │    cursor.execute(query)           [View source]│ │    │
│  │  │  psycopg2.OperationalError: connection refused  │ │    │
│  │  └────────────────────────────────────────────────┘ │    │
│  │                                                      │    │
│  │  Recent samples: (individual log entries)            │    │
│  │  [View in Log Explorer]                              │    │
│  │                                                      │    │
│  │  Actions: [Resolve] [Mute] [Link to issue tracker]  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Supported Languages

| Language | Stack Trace Format | Notes |
|----------|--------------------|-------|
| **Python** | Standard traceback format | Auto-detected |
| **Java** | `at com.example.Class.method(File.java:42)` | Auto-detected |
| **Go** | `goroutine N [running]:` + stack frames | Auto-detected |
| **Node.js** | `at Function.method (file.js:42:10)` | Auto-detected |
| **Ruby** | `file.rb:42:in 'method'` | Auto-detected |
| **PHP** | `#0 file.php(42): method()` | Auto-detected |
| **C# / .NET** | `at Namespace.Class.Method() in file.cs:line 42` | Auto-detected |

---

## Part 3 — Error Notifications

### Notification Settings

```
┌──────────────────────────────────────────────────────────────┐
│              ERROR NOTIFICATIONS                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Console → Error Reporting → Settings (gear icon)             │
│                                                                │
│  Notification triggers:                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ☑ New error group detected                          │    │
│  │    → First time this error has ever been seen        │    │
│  │                                                      │    │
│  │  ☑ Error group regressed                             │    │
│  │    → Previously resolved error has recurred          │    │
│  │                                                      │    │
│  │  ☑ Frequency threshold exceeded                      │    │
│  │    → Error count > N in time window                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Notification channels:                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ☑ Email (project owners by default)                 │    │
│  │  ☑ Google Cloud Console mobile app                   │    │
│  │  ☐ Cloud Monitoring notification channels            │    │
│  │    (Slack, PagerDuty, Pub/Sub, webhook, etc.)        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  For Pub/Sub integration (programmatic handling):             │
│  • Error notification → Pub/Sub → Cloud Function             │
│  • Cloud Function creates Jira/GitHub issue automatically    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Configuring Notifications

```bash
# Error Reporting uses Cloud Monitoring notification channels
# for alerts beyond email. Set up the channels first:

# Create Slack notification channel (in Cloud Monitoring)
gcloud alpha monitoring channels create \
    --display-name="Error Reporting Slack" \
    --type=slack \
    --channel-labels=channel_name="#app-errors"

# Then configure Error Reporting to use it via Console:
# Console → Error Reporting → Settings → Notifications
# Select the notification channels to use
```

---

## Part 4 — Reporting Errors from Applications

### Automatic Error Detection (via Cloud Logging)

Error Reporting automatically parses errors from log entries with:
- `severity = ERROR` (or higher)
- A recognisable stack trace or exception in the payload

```python
# Python — errors detected automatically when logged
import logging

logging.basicConfig()
logger = logging.getLogger(__name__)

try:
    result = process_order(order_id)
except Exception:
    logger.exception("Failed to process order")
    # logger.exception() logs the full stack trace
    # Error Reporting automatically picks it up
```

### Structured Error Logging

```python
# Python — structured logging with proper fields for Error Reporting
import json
import traceback
import sys

def report_error(message, **context):
    """Write a structured error that Error Reporting can parse."""
    exc_info = sys.exc_info()
    entry = {
        "severity": "ERROR",
        "message": message,
        **context,
    }
    if exc_info[0] is not None:
        entry["stack_trace"] = traceback.format_exc()
    
    # For Cloud Run / GKE — write to stderr as JSON
    print(json.dumps(entry), file=sys.stderr)

# Usage
try:
    charge_payment(order)
except PaymentError as e:
    report_error("Payment failed",
        orderId=order.id,
        amount=order.total,
        error=str(e),
    )
```

```go
// Go — structured error for Error Reporting
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "runtime/debug"
)

type ErrorEntry struct {
    Severity   string `json:"severity"`
    Message    string `json:"message"`
    StackTrace string `json:"stack_trace,omitempty"`
}

func reportError(msg string) {
    entry := ErrorEntry{
        Severity:   "ERROR",
        Message:    msg,
        StackTrace: string(debug.Stack()),
    }
    data, _ := json.Marshal(entry)
    fmt.Fprintln(os.Stderr, string(data))
}
```

```javascript
// Node.js — structured error for Error Reporting
function reportError(message, error, context = {}) {
  const entry = {
    severity: 'ERROR',
    message: message,
    stack_trace: error ? error.stack : new Error(message).stack,
    ...context,
  };
  console.error(JSON.stringify(entry));
}

// Usage
try {
  await processPayment(order);
} catch (err) {
  reportError('Payment processing failed', err, {
    orderId: order.id,
    amount: order.total,
  });
}
```

### Using the Error Reporting Client Library

```python
# pip install google-cloud-error_reporting
from google.cloud import error_reporting

client = error_reporting.Client(
    service="checkout-api",
    version="2.1.0",
)

# Report an exception
try:
    process_order(order_id)
except Exception:
    client.report_exception()

# Report a custom error message
client.report("Payment gateway timeout after 30s")
```

```javascript
// npm install @google-cloud/error-reporting
const { ErrorReporting } = require('@google-cloud/error-reporting');

const errors = new ErrorReporting({
  projectId: 'my-project',
  serviceContext: {
    service: 'checkout-api',
    version: '2.1.0',
  },
});

// Report an error
app.get('/checkout', async (req, res) => {
  try {
    await processOrder(req.body);
    res.json({ status: 'ok' });
  } catch (err) {
    errors.report(err);
    res.status(500).json({ error: 'Internal error' });
  }
});

// Express error handler middleware
app.use(errors.express);
```

### Service Context

```
Error Reporting groups errors by service and version.
Always set the service context for proper grouping.

For Cloud Run / App Engine / Cloud Functions:
  → Service context is auto-detected from environment

For GKE / Compute Engine:
  → Set explicitly in your application code or via
    the "serviceContext" field in structured logs:

{
  "severity": "ERROR",
  "message": "...",
  "stack_trace": "...",
  "serviceContext": {
    "service": "checkout-api",
    "version": "2.1.0"
  }
}
```

---

## Part 5 — Error Reporting for GCP Services

### Auto-Detected Errors by Service

| Service | How Errors Are Detected | Configuration |
|---------|------------------------|---------------|
| **Cloud Run** | stderr logs with stack traces | Automatic |
| **App Engine** | Request logs + application logs | Automatic |
| **Cloud Functions** | Function execution errors | Automatic |
| **GKE** | Container stderr logs with stack traces | Automatic (if logging enabled) |
| **Compute Engine** | Ops Agent log collection | Requires Ops Agent |
| **Cloud SQL** | Database error logs | Automatic (if logging enabled) |

### Cloud Run Error Reporting

```python
# Cloud Run — errors in stderr are automatically detected
# No special setup needed

from flask import Flask
app = Flask(__name__)

@app.route('/checkout', methods=['POST'])
def checkout():
    try:
        process_order(request.json)
        return {'status': 'ok'}, 200
    except Exception as e:
        # This stack trace in stderr → Error Reporting
        import traceback
        traceback.print_exc()
        return {'error': 'Internal error'}, 500
```

### App Engine Error Reporting

```python
# App Engine — automatic for uncaught exceptions
# Also works with logging.exception()

import logging
from flask import Flask

app = Flask(__name__)

@app.route('/api/order')
def handle_order():
    try:
        return process()
    except ValueError as e:
        logging.exception("Invalid order data")
        # ^ automatically appears in Error Reporting
        return "Bad request", 400
```

---

## Part 6 — Error Reporting API

### Listing Error Groups

```bash
# List error groups (recent errors)
curl -X GET \
  "https://clouderrorreporting.googleapis.com/v1beta1/projects/my-project/groupStats" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -G --data-urlencode "timeRange.period=PERIOD_1_HOUR"
```

### Reporting Errors via API

```bash
# Report an error event directly via API
curl -X POST \
  "https://clouderrorreporting.googleapis.com/v1beta1/projects/my-project/events:report" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "serviceContext": {
      "service": "checkout-api",
      "version": "2.1.0"
    },
    "message": "java.lang.NullPointerException: order is null\n\tat com.example.OrderService.process(OrderService.java:42)\n\tat com.example.API.handleCheckout(API.java:15)",
    "context": {
      "httpRequest": {
        "method": "POST",
        "url": "/api/checkout",
        "responseStatusCode": 500
      },
      "reportLocation": {
        "filePath": "OrderService.java",
        "lineNumber": 42,
        "functionName": "process"
      }
    }
  }'
```

### Deleting Error Events

```bash
# Delete all error events for a project (reset)
curl -X DELETE \
  "https://clouderrorreporting.googleapis.com/v1beta1/projects/my-project/events" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)"
```

---

## Part 7 — Resolution & Muting

### Error Lifecycle

```
┌──────────────────────────────────────────────────────────────┐
│              ERROR LIFECYCLE                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────┐     ┌──────────┐     ┌──────────┐                 │
│  │ NEW  │────►│  OPEN    │────►│ RESOLVED │                 │
│  │ 🆕   │     │  ⚠       │     │  ✓       │                 │
│  └──────┘     └──────────┘     └─────┬────┘                 │
│                    ▲                  │                        │
│                    │                  │ error recurs           │
│                    │                  ▼                        │
│                    │           ┌──────────┐                   │
│                    └───────────│ REGRESSED│                   │
│                               │  🔄      │                   │
│                               └──────────┘                   │
│                                                                │
│  States:                                                       │
│  • NEW: First-ever occurrence of this error                  │
│  • OPEN: Active error group receiving events                 │
│  • RESOLVED: Manually marked as fixed                        │
│  • REGRESSED: Was resolved, but new occurrences appeared     │
│                                                                │
│  Muting:                                                       │
│  • Muted errors don't trigger notifications                  │
│  • Still visible in dashboard (dimmed)                       │
│  • Useful for known issues or expected errors                │
│  • Can be un-muted at any time                               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Managing Error State

```
Console actions:
• Click error group → [Resolve] button → marks as resolved
• Click error group → [Mute] → suppresses notifications
• If resolved error recurs → automatically reopens + notifies

Best practices:
• Resolve errors after deploying a fix
• Mute expected errors (e.g., client validation errors)
• Use resolution to track fix verification
• Link to issue tracker (GitHub/Jira URL) for tracking
```

---

## Part 8 — Cloud Profiler Fundamentals

### What Is Cloud Profiler?

Cloud Profiler is a continuous production profiler that collects CPU, heap, and wall-clock data from your running applications with minimal overhead (~0.5% CPU):

```
┌─────────────────────────────────────────────────────────────────┐
│               CLOUD PROFILER OVERVIEW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Your Application (production)                            │   │
│  │                                                           │   │
│  │  ┌────────────────┐                                      │   │
│  │  │ Profiler Agent  │  ← embedded in your app             │   │
│  │  │                │  ← samples stack traces every 10ms   │   │
│  │  │                │  ← collects for ~10s every 10min     │   │
│  │  └───────┬────────┘                                      │   │
│  └──────────┼───────────────────────────────────────────────┘   │
│             │ upload profile                                     │
│             ▼                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Cloud Profiler Backend                                   │   │
│  │                                                           │   │
│  │  ┌────────────┐  ┌────────────┐  ┌─────────────────┐   │   │
│  │  │ Flame Graph │  │ Compare    │  │ Filter by       │   │   │
│  │  │ View       │  │ Versions   │  │ Service/Version │   │   │
│  │  └────────────┘  └────────────┘  └─────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Benefits:                                                        │
│  • Identify CPU/memory hot spots in production                   │
│  • Compare performance across deployments                        │
│  • Very low overhead (~0.5% CPU)                                 │
│  • No code changes needed (agent-based)                          │
│  • Always-on in production (not just during incidents)           │
│  • Free of charge                                                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Cloud Profiler | AWS CodeGuru Profiler | Azure Application Insights Profiler |
|---------|-------------------|----------------------|--------------------------------------|
| Service | Cloud Profiler | CodeGuru Profiler | App Insights Profiler |
| Overhead | ~0.5% CPU | ~1% CPU | ~5% (when profiling) |
| Languages | Python, Java, Go, Node.js | Java, Python | .NET, Java |
| Profile types | CPU, heap, wall, threads, contention | CPU, heap, latency | CPU sampling |
| Continuous | Yes (always on) | Yes | Triggered or scheduled |
| Flame graphs | Yes | Yes | Yes |
| Version comparison | Yes | Recommendations | Limited |
| Pricing | **Free** | ~$0.005/agent-hour | Included in App Insights |
| Agent | Embedded library | Agent JAR / module | Site extension / SDK |

### Pricing

| Component | Cost |
|-----------|------|
| Cloud Profiler | **Free** |
| Profile storage (30 days) | Free |
| Agent overhead | ~0.5% CPU |
| API calls | Free |

---

## Part 9 — Profiler Architecture & Profiling Types

### How Profiling Works

```
┌──────────────────────────────────────────────────────────────┐
│              PROFILING MECHANISM                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Statistical sampling (not instrumentation):                   │
│                                                                │
│  Time ──────────────────────────────────────────────────►     │
│                                                                │
│  ┌─────┐           ┌─────┐           ┌─────┐               │
│  │Prof │  (pause)  │Prof │  (pause)  │Prof │               │
│  │10s  │  ~10min   │10s  │  ~10min   │10s  │               │
│  └─────┘           └─────┘           └─────┘               │
│                                                                │
│  During each 10-second profiling window:                      │
│  • Sample stack traces at ~100 Hz (every 10ms)               │
│  • Record which function is at the top of the stack          │
│  • Aggregate: function X appeared in N% of samples           │
│  • Upload aggregated profile to Profiler backend             │
│                                                                │
│  Result interpretation:                                        │
│  If function "db_query()" appears in 45% of CPU samples,     │
│  then ~45% of CPU time is spent in that function.            │
│                                                                │
│  Low overhead because:                                         │
│  • Only profiles 10s out of every ~10 min                    │
│  • Statistical sampling, not tracing every call              │
│  • Runs on one replica at a time (not all replicas)          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Profile Types

| Profile Type | Description | Languages |
|-------------|-------------|-----------|
| **CPU time** | Time the CPU spends executing your code (excludes I/O waits) | All |
| **Wall time** | Elapsed real time (includes I/O, network, sleep) | Python, Java, Go, Node.js |
| **Heap** | Memory allocated on the heap at a point in time | Java, Go, Python |
| **Allocated heap** | Total memory allocated (including freed) over the profiling window | Java, Go |
| **Threads** | Number of threads at a point in time | Java |
| **Contention** | Time threads spend waiting for locks | Java, Go |

### When to Use Each Profile Type

| Scenario | Profile Type | What to Look For |
|----------|-------------|------------------|
| High CPU usage | CPU time | Functions consuming most CPU cycles |
| Slow responses (not CPU-bound) | Wall time | I/O waits, network calls, sleep |
| Memory leaks | Heap | Growing allocations over time |
| Excessive garbage collection | Allocated heap | High allocation rate |
| Thread starvation | Threads + Contention | Lock contention, blocked threads |
| External API bottleneck | Wall time | Long wall time with low CPU time |

---

## Part 10 — Instrumenting Applications for Profiler

### Python

```python
# pip install google-cloud-profiler

import googlecloudprofiler

def main():
    # Start profiler (call once at application startup)
    googlecloudprofiler.start(
        service='checkout-api',
        service_version='2.1.0',
        verbose=0,
        # project_id='my-project',  # auto-detected on GCP
    )
    
    # Rest of your application...
    app.run(host='0.0.0.0', port=8080)

if __name__ == '__main__':
    main()
```

### Java

```java
// Add dependency:
// com.google.cloud:google-cloud-profiler:latest

// Option 1: Agent flag (recommended — zero code changes)
// java -agentpath:/opt/cprof/profiler_java_agent.so=\
//   -cprof_service=checkout-api,\
//   -cprof_service_version=2.1.0 \
//   -jar myapp.jar

// Option 2: Programmatic
import com.google.devtools.cloudprofiler.v2.ProfilerAgent;

public class Application {
    public static void main(String[] args) throws Exception {
        ProfilerAgent.start(
            ProfilerAgent.Config.builder()
                .setService("checkout-api")
                .setVersion("2.1.0")
                .build()
        );
        // Start application...
    }
}
```

### Go

```go
// go get cloud.google.com/go/profiler

package main

import (
    "cloud.google.com/go/profiler"
    "log"
)

func main() {
    // Start profiler
    cfg := profiler.Config{
        Service:        "checkout-api",
        ServiceVersion: "2.1.0",
        // ProjectID:   "my-project", // auto-detected on GCP
        
        // Enable all profile types
        MutexProfiling: true,
    }
    if err := profiler.Start(cfg); err != nil {
        log.Printf("Failed to start profiler: %v", err)
        // Don't crash — profiling is optional
    }

    // Rest of your application...
}
```

### Node.js

```javascript
// npm install @google-cloud/profiler

// Must be the FIRST require in your application
require('@google-cloud/profiler').start({
  serviceContext: {
    service: 'checkout-api',
    version: '2.1.0',
  },
  // projectId: 'my-project',  // auto-detected on GCP
});

// Rest of your application...
const express = require('express');
const app = express();
// ...
```

### Docker / Cloud Run Setup

```dockerfile
# Python example
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Profiler starts in application code (see Python example above)
CMD ["python", "main.py"]
```

```yaml
# Cloud Run service — no special config needed
# Just include the profiler library and start it in code
# The service account needs roles/cloudprofiler.agent
```

---

## Part 11 — Profiler Console & Analysis

### Profiler Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│              CLOUD PROFILER CONSOLE                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Console → Profiler                                           │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Service: [checkout-api     ▼]                       │    │
│  │  Profile: [CPU time         ▼]                       │    │
│  │  Version: [2.1.0            ▼]                       │    │
│  │  Zone:    [All zones        ▼]                       │    │
│  │  Time:    [Last 24 hours    ▼]                       │    │
│  │  Weight:  [Focus ▼]  Filter: [____________]          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Flame Graph (CPU Time)                               │    │
│  │                                                      │    │
│  │  ┌─────────────────────────────────────────────────┐│    │
│  │  │ main() ████████████████████████████████████████ ││    │
│  │  │ ├─ handle_request() ████████████████████████   ││    │
│  │  │ │  ├─ process_order() █████████████████        ││    │
│  │  │ │  │  ├─ db_query() ████████████  (35%)       ││    │
│  │  │ │  │  ├─ validate() █████  (12%)              ││    │
│  │  │ │  │  └─ payment() ████████  (22%)            ││    │
│  │  │ │  │     └─ http_post() ██████  (18%)         ││    │
│  │  │ │  └─ serialize() ███  (8%)                   ││    │
│  │  │ └─ middleware() ████  (10%)                    ││    │
│  │  └─────────────────────────────────────────────────┘│    │
│  │                                                      │    │
│  │  → db_query() consumes 35% of CPU → optimise it!    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Reading the Flame Graph

```
┌──────────────────────────────────────────────────────────────┐
│         HOW TO READ A FLAME GRAPH                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  • Width = percentage of total resource usage                 │
│  • Y-axis = call stack depth (parent above child)            │
│  • Color = visual grouping (by package/module)               │
│                                                                │
│  Interpretation:                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ main()        ████████████████████████████████████  │     │
│  │                    Wide = this function (and         │     │
│  │                    its children) use a lot of time   │     │
│  │                                                     │     │
│  │ db_query()    ████████████                          │     │
│  │                    Tall bar at bottom with no       │     │
│  │                    children = "self time" (the      │     │
│  │                    function itself is doing work)   │     │
│  │                                                     │     │
│  │ validate()    █████                                 │     │
│  │                    Narrow = small percentage        │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                                │
│  Key insight:                                                  │
│  Look for WIDE bars at the BOTTOM of the graph              │
│  → These are the actual hot functions consuming resources    │
│                                                                │
│  Interactions:                                                 │
│  • Click a function to zoom in                                │
│  • Hover for exact percentage                                 │
│  • Use "Focus" to highlight a specific function path         │
│  • Use "Filter" to search for function names                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Profile Metrics Table

Below the flame graph, Profiler shows a sortable table:

| Function | Self (%) | Total (%) | Self (ms) |
|----------|----------|-----------|-----------|
| `db_query()` | 35.2% | 35.2% | 3,520 |
| `payment.http_post()` | 18.1% | 18.1% | 1,810 |
| `validate()` | 12.0% | 12.0% | 1,200 |
| `process_order()` | 2.3% | 67.3% | 230 |
| `json.dumps()` | 8.5% | 8.5% | 850 |
| `middleware.auth()` | 6.2% | 10.0% | 620 |

> **Self** = time in the function itself. **Total** = time including all child calls.

---

## Part 12 — Comparing Profiles & Flame Graphs

### Version Comparison

```
┌──────────────────────────────────────────────────────────────┐
│           PROFILE COMPARISON                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Compare v2.0.0 (baseline) vs v2.1.0 (current):              │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Comparison Flame Graph                               │    │
│  │                                                      │    │
│  │  Colors indicate change:                              │    │
│  │  🟦 Blue = got FASTER (less resource usage)          │    │
│  │  🟥 Red  = got SLOWER (more resource usage)          │    │
│  │  ⬜ Grey = unchanged                                  │    │
│  │                                                      │    │
│  │  ┌─────────────────────────────────────────────┐    │    │
│  │  │ main() ████████████████████████████████████ │    │    │
│  │  │ ├─ handle_request() ██████████████████████ │    │    │
│  │  │ │  ├─ db_query() 🟦🟦🟦🟦🟦🟦  (-15%)   │    │    │
│  │  │ │  │  └─ (query cache added in v2.1!)     │    │    │
│  │  │ │  ├─ validate() ⬜⬜⬜  (unchanged)      │    │    │
│  │  │ │  └─ payment() 🟥🟥🟥🟥🟥🟥🟥  (+25%) │    │    │
│  │  │ │     └─ new_fraud_check() 🟥🟥🟥        │    │    │
│  │  │ │        (new in v2.1 — takes 25% more)   │    │    │
│  │  └─┴────────────────────────────────────────────┘    │    │
│  │                                                      │    │
│  │  Summary:                                            │    │
│  │  • db_query() improved 15% (query cache effective)   │    │
│  │  • payment() regressed 25% (new fraud check)         │    │
│  │  • Net change: +5% CPU overall                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Console → Profiler → Compare to: [v2.0.0 ▼]                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Comparison Workflow

```
1. Deploy new version with updated service_version
2. Wait for profiles to collect (10-30 minutes)
3. Console → Profiler → select service
4. Set "Compare to" dropdown to previous version
5. Review comparison flame graph:
   • Blue areas = improvements (celebrate!)
   • Red areas = regressions (investigate)
6. Zoom into red areas to find root cause
7. Fix and redeploy if needed
```

### Time-Based Comparison

```
You can also compare profiles across time windows:
• "Last 1 hour" vs "Previous 24 hours"
• Useful for detecting gradual performance degradation
• Helps identify impact of external factors
  (e.g., increased load, database growth)
```

---

## Part 13 — IAM & Security

### IAM Roles

| Role | Error Reporting | Cloud Profiler |
|------|----------------|----------------|
| `roles/errorreporting.viewer` | View errors | — |
| `roles/errorreporting.writer` | Report errors | — |
| `roles/errorreporting.admin` | Full control (view + write + delete) | — |
| `roles/cloudprofiler.agent` | — | Upload profiles |
| `roles/cloudprofiler.user` | — | View profiles |

### Granting Permissions

```bash
# ─── Error Reporting ─────────────────────────────────────────
# App service account — write errors
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:app-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/errorreporting.writer"

# Developers — view errors
gcloud projects add-iam-policy-binding my-project \
    --member="group:developers@example.com" \
    --role="roles/errorreporting.viewer"

# ─── Cloud Profiler ──────────────────────────────────────────
# App service account — upload profiles
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:app-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/cloudprofiler.agent"

# Developers — view profiles
gcloud projects add-iam-policy-binding my-project \
    --member="group:developers@example.com" \
    --role="roles/cloudprofiler.user"
```

### Security Considerations

```
┌──────────────────────────────────────────────────────────────┐
│         SECURITY BEST PRACTICES                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Error Reporting:                                              │
│  • Stack traces may contain file paths, variable names       │
│  • Don't include sensitive data in error messages            │
│  • Don't log PII, credentials, or tokens in exceptions      │
│  • Use errorreporting.viewer (not admin) for developers     │
│                                                                │
│  Cloud Profiler:                                               │
│  • Profiles contain function names, call stacks              │
│  • No actual data values are captured (just timing)          │
│  • Restrict cloudprofiler.user to trusted users              │
│  • Profiles could reveal internal architecture               │
│  • Use Workload Identity for GKE (not node SA)              │
│                                                                │
│  General:                                                      │
│  • Both services auto-detect on GCP (no key management)     │
│  • For non-GCP environments, use service account keys       │
│  • Enable APIs only in projects that need them               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Enable APIs

```hcl
# Enable Error Reporting API
resource "google_project_service" "error_reporting" {
  project = var.project_id
  service = "clouderrorreporting.googleapis.com"
}

# Enable Cloud Profiler API
resource "google_project_service" "profiler" {
  project = var.project_id
  service = "cloudprofiler.googleapis.com"
}
```

### Terraform — IAM for Error Reporting & Profiler

```hcl
# Service account for applications
resource "google_service_account" "app_sa" {
  account_id   = "app-observability-sa"
  display_name = "App Observability SA"
  project      = var.project_id
}

# Error Reporting — write errors
resource "google_project_iam_member" "error_writer" {
  project = var.project_id
  role    = "roles/errorreporting.writer"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

# Cloud Profiler — upload profiles
resource "google_project_iam_member" "profiler_agent" {
  project = var.project_id
  role    = "roles/cloudprofiler.agent"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

# Cloud Trace — write traces (often needed together)
resource "google_project_iam_member" "trace_agent" {
  project = var.project_id
  role    = "roles/cloudtrace.agent"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

# Cloud Logging — write logs
resource "google_project_iam_member" "log_writer" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

# Developers — view all observability data
resource "google_project_iam_member" "error_viewer" {
  project = var.project_id
  role    = "roles/errorreporting.viewer"
  member  = "group:developers@example.com"
}

resource "google_project_iam_member" "profiler_viewer" {
  project = var.project_id
  role    = "roles/cloudprofiler.user"
  member  = "group:developers@example.com"
}
```

### Terraform — Cloud Run with Full Observability

```hcl
# Cloud Run service with Error Reporting + Profiler + Trace
resource "google_cloud_run_v2_service" "api" {
  name     = "checkout-api"
  location = "us-central1"
  project  = var.project_id

  template {
    service_account = google_service_account.app_sa.email

    containers {
      image = "gcr.io/${var.project_id}/checkout-api:${var.app_version}"

      env {
        name  = "SERVICE_NAME"
        value = "checkout-api"
      }
      env {
        name  = "SERVICE_VERSION"
        value = var.app_version
      }
      env {
        name  = "GOOGLE_CLOUD_PROJECT"
        value = var.project_id
      }
      env {
        name  = "ENABLE_PROFILER"
        value = "true"
      }

      resources {
        limits = {
          cpu    = "2"
          memory = "1Gi"
        }
      }
    }

    scaling {
      min_instance_count = 1
      max_instance_count = 10
    }
  }
}
```

### Terraform — Monitoring Alert on Error Count

```hcl
# Alert when error count spikes (uses log-based metric)
resource "google_logging_metric" "app_errors" {
  name    = "app_error_count"
  project = var.project_id
  filter  = "resource.type = \"cloud_run_revision\" AND severity >= ERROR AND resource.labels.service_name = \"checkout-api\""

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
    unit        = "1"
  }
}

resource "google_monitoring_alert_policy" "error_spike" {
  display_name = "Error Count Spike"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "Error count > 50 in 5 minutes"

    condition_threshold {
      filter          = "metric.type = \"logging.googleapis.com/user/app_error_count\" AND resource.type = \"cloud_run_revision\""
      comparison      = "COMPARISON_GT"
      threshold_value = 50
      duration        = "300s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_SUM"
      }
    }
  }

  notification_channels = var.notification_channel_ids
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# ERROR REPORTING
# ═══════════════════════════════════════════════════════════════

# Enable API
gcloud services enable clouderrorreporting.googleapis.com

# View errors (via Console — no gcloud CLI for listing)
# Console → Error Reporting

# Delete all error events (reset)
gcloud beta error-reporting events delete --project=my-project

# Grant error writer role
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:SA_EMAIL" \
    --role="roles/errorreporting.writer"

# Grant error viewer role
gcloud projects add-iam-policy-binding my-project \
    --member="user:dev@example.com" \
    --role="roles/errorreporting.viewer"

# ═══════════════════════════════════════════════════════════════
# CLOUD PROFILER
# ═══════════════════════════════════════════════════════════════

# Enable API
gcloud services enable cloudprofiler.googleapis.com

# View profiles (via Console — no gcloud CLI for viewing)
# Console → Profiler

# Grant profiler agent role (for applications)
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:SA_EMAIL" \
    --role="roles/cloudprofiler.agent"

# Grant profiler user role (for developers)
gcloud projects add-iam-policy-binding my-project \
    --member="group:developers@example.com" \
    --role="roles/cloudprofiler.user"

# ═══════════════════════════════════════════════════════════════
# COMBINED OBSERVABILITY SA SETUP
# ═══════════════════════════════════════════════════════════════

# Create a service account with all observability permissions
gcloud iam service-accounts create app-observability \
    --display-name="App Observability SA" \
    --project=my-project

SA="app-observability@my-project.iam.gserviceaccount.com"

# Grant all observability roles
for ROLE in \
    roles/logging.logWriter \
    roles/cloudtrace.agent \
    roles/monitoring.metricWriter \
    roles/errorreporting.writer \
    roles/cloudprofiler.agent; do
  gcloud projects add-iam-policy-binding my-project \
      --member="serviceAccount:$SA" \
      --role="$ROLE"
done
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Full Observability Stack for Microservices

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: COMPLETE OBSERVABILITY PIPELINE                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Every microservice includes:                                          │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  Application Code                                          │      │
│  │  ├── Cloud Profiler agent (CPU + heap profiling)          │      │
│  │  ├── OpenTelemetry SDK (traces + trace-log correlation)   │      │
│  │  ├── Structured logging (JSON to stderr)                  │      │
│  │  └── Error reporting (via logging or client library)      │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Data flows:                                                           │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  Application                                               │      │
│  │  │                                                         │      │
│  │  ├──→ Cloud Logging (structured logs with trace IDs)      │      │
│  │  │    └──→ Error Reporting (auto-parses stack traces)     │      │
│  │  │    └──→ Log-based metrics (error count, latency)       │      │
│  │  │                                                         │      │
│  │  ├──→ Cloud Trace (distributed traces, latency)           │      │
│  │  │    └──→ Correlated with logs (same trace ID)           │      │
│  │  │                                                         │      │
│  │  ├──→ Cloud Monitoring (metrics, dashboards, alerts)      │      │
│  │  │    └──→ SLOs + burn rate alerts                        │      │
│  │  │                                                         │      │
│  │  └──→ Cloud Profiler (continuous CPU/heap profiling)      │      │
│  │       └──→ Version comparison on each deploy              │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Incident workflow:                                                    │
│  1. Alert fires (Cloud Monitoring — error rate > 5%)                  │
│  2. Check Error Reporting — what's the new error?                     │
│  3. Click stack trace → view logs → find specific failure             │
│  4. Click trace ID → see full request flow in Cloud Trace            │
│  5. Identify bottleneck span → correlate with profiler               │
│  6. Cloud Profiler → compare current vs previous version             │
│  7. Fix → deploy → verify error count drops to zero                  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation (Python/Cloud Run):**

```python
# main.py — Full observability setup for a Cloud Run service
import os
import json
import sys

# 1. Start Cloud Profiler (must be before other imports)
import googlecloudprofiler
try:
    googlecloudprofiler.start(
        service=os.getenv('SERVICE_NAME', 'my-service'),
        service_version=os.getenv('SERVICE_VERSION', '1.0.0'),
        verbose=0,
    )
except Exception as e:
    print(f"Profiler init failed: {e}", file=sys.stderr)

# 2. Set up OpenTelemetry tracing
from opentelemetry import trace, propagate
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.propagators.cloud_trace_propagator import CloudTraceFormatPropagator
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.flask import FlaskInstrumentor

resource = Resource.create({
    "service.name": os.getenv('SERVICE_NAME', 'my-service'),
    "service.version": os.getenv('SERVICE_VERSION', '1.0.0'),
})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter()))
trace.set_tracer_provider(provider)
propagate.set_global_textmap(CloudTraceFormatPropagator())
RequestsInstrumentor().instrument()

# 3. Structured logging with trace correlation
tracer = trace.get_tracer(__name__)
PROJECT = os.getenv('GOOGLE_CLOUD_PROJECT', '')

def log(severity, message, **kwargs):
    entry = {"severity": severity, "message": message, **kwargs}
    span = trace.get_current_span()
    ctx = span.get_span_context()
    if ctx.is_valid:
        entry["logging.googleapis.com/trace"] = f"projects/{PROJECT}/traces/{format(ctx.trace_id, '032x')}"
        entry["logging.googleapis.com/spanId"] = format(ctx.span_id, '016x')
    print(json.dumps(entry), file=sys.stderr if severity in ('ERROR','CRITICAL') else sys.stdout)

# 4. Flask app with auto-instrumentation
from flask import Flask, request
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

@app.route('/api/checkout', methods=['POST'])
def checkout():
    with tracer.start_as_current_span("checkout") as span:
        order_id = request.json.get('orderId')
        span.set_attribute("order.id", order_id)
        log("INFO", f"Processing checkout", orderId=order_id)
        
        try:
            result = process_order(order_id)
            log("INFO", f"Checkout complete", orderId=order_id)
            return {"status": "ok"}, 200
        except Exception as e:
            # Error automatically detected by Error Reporting
            import traceback
            log("ERROR", f"Checkout failed", orderId=order_id,
                stack_trace=traceback.format_exc())
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            span.record_exception(e)
            return {"error": "Internal error"}, 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.getenv('PORT', 8080)))
```

### Pattern 2: Performance Regression Detection Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: DEPLOY → PROFILE → COMPARE → ALERT                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Automated performance validation on every deployment:                 │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Cloud Build / Cloud Deploy Pipeline                      │        │
│  │                                                           │        │
│  │  1. Build image                                           │        │
│  │  2. Deploy canary (10% traffic)                          │        │
│  │  3. Wait 15 minutes (collect profiles + traces)          │        │
│  │  4. Compare profiles:                                     │        │
│  │     • CPU regression > 10%? → FAIL                       │        │
│  │     • Heap regression > 20%? → FAIL                      │        │
│  │     • p99 latency regression > 15%? → FAIL               │        │
│  │  5. Check Error Reporting:                                │        │
│  │     • New error groups? → FAIL                           │        │
│  │     • Error count > baseline? → FAIL                      │        │
│  │  6. If all pass → promote to 100%                        │        │
│  │  7. If any fail → rollback + notify                       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Monitoring dashboard (per deployment):                                │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │ Version │ CPU (self) │ Heap Peak │ p99 Latency │ Errors    │     │
│  │ v2.0.0  │ baseline   │ 256 MB    │ 300ms       │ 2/min     │     │
│  │ v2.1.0  │ +5%        │ 280 MB    │ 340ms       │ 3/min     │     │
│  │ v2.2.0  │ +18% ⚠    │ 512 MB ⚠ │ 800ms ⚠    │ 15/min ⚠ │     │
│  │         │ ROLLBACK   │ ROLLBACK  │ ROLLBACK    │ ROLLBACK  │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Error Budget Driven by Error Reporting

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: ERROR REPORTING + SLO ERROR BUDGETS                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Combine Error Reporting with Cloud Monitoring SLOs:                  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Error Reporting Dashboard                                │        │
│  │                                                           │        │
│  │  checkout-api v2.1.0                                      │        │
│  │  ┌──────────────────────────┬─────────────────────────┐  │        │
│  │  │ Error Group              │ Count (24h) │ Status    │  │        │
│  │  │ PaymentTimeout           │ 45          │ 🆕 NEW   │  │        │
│  │  │ DBConnectionRefused      │ 12          │ ⚠ OPEN   │  │        │
│  │  │ ValidationError          │ 230         │ Muted    │  │        │
│  │  │ NullPointerException     │ 0           │ ✓ Resolved│ │        │
│  │  └──────────────────────────┴─────────────────────────┘  │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  SLO impact:                                                           │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  SLO: 99.9% availability (30-day rolling)                │        │
│  │  Error budget: 43.2 minutes                               │        │
│  │  Consumed: 12.5 minutes (29%)                             │        │
│  │                                                           │        │
│  │  PaymentTimeout (45 errors) = ~3 minutes of budget       │        │
│  │  DBConnectionRefused (12 errors) = ~1 minute of budget   │        │
│  │  ValidationError (230) = not counted (4xx, user error)   │        │
│  │                                                           │        │
│  │  Action: Fix PaymentTimeout before budget runs out       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Workflow:                                                             │
│  1. New error appears in Error Reporting → notification               │
│  2. Check SLO dashboard → how much budget is it consuming?           │
│  3. If budget burn rate is high → immediate fix (page on-call)       │
│  4. If budget burn rate is low → schedule fix (create ticket)        │
│  5. Use Cloud Profiler to identify if error is performance-related   │
│  6. Use Cloud Trace to trace the failing request path                │
│  7. Fix, deploy, verify error resolves in Error Reporting            │
│  8. Confirm SLO burn rate returns to normal                          │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Error Reporting

| Action | Command / Location |
|--------|-------------------|
| View errors | Console → Error Reporting |
| Enable API | `gcloud services enable clouderrorreporting.googleapis.com` |
| Delete all events | `gcloud beta error-reporting events delete` |
| Grant write access | `--role="roles/errorreporting.writer"` |
| Grant view access | `--role="roles/errorreporting.viewer"` |
| Python client | `pip install google-cloud-error_reporting` |
| Node.js client | `npm install @google-cloud/error-reporting` |
| Pricing | **Free** |

### Cloud Profiler

| Action | Command / Location |
|--------|-------------------|
| View profiles | Console → Profiler |
| Enable API | `gcloud services enable cloudprofiler.googleapis.com` |
| Grant agent role | `--role="roles/cloudprofiler.agent"` |
| Grant viewer role | `--role="roles/cloudprofiler.user"` |
| Python agent | `pip install google-cloud-profiler` |
| Go agent | `go get cloud.google.com/go/profiler` |
| Node.js agent | `npm install @google-cloud/profiler` |
| Java agent | `-agentpath:/opt/cprof/profiler_java_agent.so` |
| Profile types | CPU, Wall, Heap, Allocated Heap, Threads, Contention |
| Overhead | ~0.5% CPU |
| Pricing | **Free** |
| Retention | 30 days |

---

## What is Profiling? (Beginner Explanation)

### The Speedometer Analogy

Think of **profiling as a speedometer for your code** — it shows which functions are slow and using too much memory.

```
┌──────────────────────────────────────────────────────────────┐
│         PROFILING = SPEEDOMETER FOR YOUR CODE                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Car Dashboard               Code Profiler                    │
│  ─────────────               ──────────────                   │
│                                                                │
│  Speedometer          ═══►   CPU Profile                      │
│  (how fast engine               (how much processing power    │
│   is working)                    each function uses)           │
│                                                                │
│  Fuel gauge           ═══►   Heap (Memory) Profile            │
│  (how much fuel                 (how much memory each         │
│   is being consumed)             object is using)              │
│                                                                │
│  RPM meter            ═══►   Wall-Clock Profile               │
│  (engine revolutions           (total elapsed time including  │
│   per minute)                   waiting for I/O)               │
│                                                                │
│  Temperature warning  ═══►   Hot function                     │
│  (engine overheating)          (function consuming too         │
│                                 much CPU or memory)            │
│                                                                │
│  Diagnostic port      ═══►   Profiling agent                  │
│  (plug in to read data)        (lightweight agent in your     │
│                                 app collecting samples)        │
│                                                                │
│  Mechanic report      ═══►   Flame graph                      │
│  (shows what's wrong)          (visual showing which           │
│                                 call chains are slowest)       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Breaking It Down

| Concept | Car Analogy | What It Really Means |
|---------|-------------|---------------------|
| **CPU profile** | Speedometer showing engine effort | Shows which functions consume the most processing time |
| **Heap profile** | Fuel gauge showing consumption | Shows which objects are consuming the most memory |
| **Wall-clock profile** | Trip timer (total travel time) | Total time including waiting for network, disk, and I/O |
| **Hot function** | The part of the engine running hottest | The function that uses the most CPU — the one to optimize first |
| **Flame graph** | A map showing which engine parts connect and which are hottest | A stacked chart where wider bars = more time/memory consumed |
| **Profiling agent** | The diagnostic sensor in your car | A tiny library added to your app that samples ~100 times/sec |
| **Overhead** | Diagnostic sensors barely affect engine performance (~0.5%) | Profiler uses less than 0.5% CPU — safe for production |

### Why Does Profiling Matter?

1. **Find the real bottleneck** — Gut feelings about "slow code" are often wrong. Profiling shows you the actual hot spots
2. **Optimize what matters** — Don't spend hours optimizing a function that only takes 1% of total CPU. Focus on the 40% one
3. **Catch memory leaks** — Heap profiles reveal objects that keep growing and never get cleaned up
4. **Production-safe** — Cloud Profiler is designed for production with negligible overhead (~0.5% CPU)
5. **Compare versions** — See if your new release is faster or slower than the previous one

> **One-liner**: If your code is a car, Cloud Profiler is the dashboard showing you exactly which part of the engine is working hardest and consuming the most fuel.

---

## Console Walkthrough: Profiler Setup & Usage

### Enabling Cloud Profiler

```
┌──────────────────────────────────────────────────────────────┐
│         ENABLE CLOUD PROFILER                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Enable the API                                        │
│  Console → APIs & Services → Library                          │
│  Search for "Cloud Profiler API" → Click → ENABLE             │
│                                                                │
│  Or via CLI:                                                   │
│  gcloud services enable cloudprofiler.googleapis.com           │
│                                                                │
│  Step 2: Add the profiling agent to your application          │
│                                                                │
│  Python:                                                       │
│  pip install google-cloud-profiler                             │
│  Add to your app's startup:                                    │
│    import googlecloudprofiler                                  │
│    googlecloudprofiler.start(                                  │
│        service='my-service', service_version='1.0.0')          │
│                                                                │
│  Node.js:                                                      │
│  npm install @google-cloud/profiler                            │
│  Add to your app:                                              │
│    require('@google-cloud/profiler').start({                   │
│        serviceContext: { service: 'my-service', version: '1'}  │
│    });                                                         │
│                                                                │
│  Go / Java: Similar agent setup (see Part 10 for details)     │
│                                                                │
│  Step 3: Grant IAM role                                        │
│  Service account needs: roles/cloudprofiler.agent              │
│                                                                │
│  ⚡ Profiles start appearing within 1-2 minutes after          │
│    deployment. No restart needed for Cloud Run.               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Viewing Flame Graphs

```
┌──────────────────────────────────────────────────────────────┐
│         VIEW FLAME GRAPHS IN CONSOLE                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Profiler                                           │
│                                                                │
│  Step 2: Select service and profile type                      │
│  • Service: Choose your application (e.g., "checkout-api")   │
│  • Profile type: CPU, Wall, Heap, or Threads                 │
│  • Time range: Select the time window                         │
│  • Version: Filter by app version (if set)                    │
│                                                                │
│  Step 3: Read the flame graph                                  │
│  • Each bar = one function in your code                       │
│  • Width = how much CPU/memory that function uses             │
│  • Stacked vertically = call chain (caller on top, callee     │
│    below)                                                      │
│  • Hover over a bar to see exact percentage                   │
│  • Click a bar to zoom into that subtree                      │
│                                                                │
│  Step 4: Find hot functions                                    │
│  • Look for the widest bars — these consume the most          │
│  • Focus on YOUR code (not library/framework internals)       │
│  • The "Focus" filter lets you search for specific functions  │
│                                                                │
│  ⚡ Tip: Use "Compare to" to see if a new version is         │
│    faster or slower than a previous one.                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Finding Hot Functions

```
┌──────────────────────────────────────────────────────────────┐
│         FIND HOT FUNCTIONS                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Open a CPU profile                                    │
│  Console → Profiler → Profile type: CPU time                  │
│                                                                │
│  Step 2: Sort by "self" time                                  │
│  • "Self" = time spent IN that function (not its children)   │
│  • "Total" = time in function + all functions it calls        │
│  • High "self" = the function itself is slow                 │
│  • High "total" but low "self" = something it calls is slow │
│                                                                │
│  Step 3: Investigate                                           │
│  • Click the hot function to see its call chain               │
│  • Check: Is it a tight loop? An expensive DB query?          │
│  • Check: Is it called too many times? (N+1 problem)          │
│                                                                │
│  Step 4: Compare profiles                                      │
│  • Click "Compare to" and select a previous time range       │
│  • Blue = faster (improvement), Red = slower (regression)     │
│  • Use this after deploying a fix to verify improvement       │
│                                                                │
│  Common hot function patterns:                                 │
│  • JSON serialization (consider msgpack or protobuf)          │
│  • Regex compilation inside loops (compile once, reuse)       │
│  • Unoptimized DB queries (add indexes, reduce N+1)           │
│  • String concatenation in loops (use StringBuilder/join)     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 40: Cloud KMS** → `40-cloud-kms.md`
