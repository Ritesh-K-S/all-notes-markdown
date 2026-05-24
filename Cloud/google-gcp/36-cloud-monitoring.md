# Chapter 36 — Cloud Monitoring

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Fundamentals](#part-1--fundamentals)
- [Part 2: Architecture & Core Concepts](#part-2--architecture--core-concepts)
- [Part 3: Metrics & Time Series](#part-3--metrics--time-series)
- [Part 4: Metrics Explorer](#part-4--metrics-explorer)
- [Part 5: Dashboards](#part-5--dashboards)
- [Part 6: Alerting Policies](#part-6--alerting-policies)
- [Part 7: Notification Channels](#part-7--notification-channels)
- [Part 8: Uptime Checks](#part-8--uptime-checks)
- [Part 9: Groups](#part-9--groups)
- [Part 10: Custom Metrics](#part-10--custom-metrics)
- [Part 11: Monitoring Query Language (MQL)](#part-11--monitoring-query-language-mql)
- [Part 12: Service Monitoring & SLOs](#part-12--service-monitoring--slos)
- [Part 13: Integration with Other Services](#part-13--integration-with-other-services)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Monitoring (formerly Stackdriver Monitoring) provides full-stack observability for your cloud infrastructure and applications. It collects metrics, events, and metadata from GCP services, hosted uptime probes, application instrumentation, and other sources. With dashboards, alerting, and SLO tracking, Cloud Monitoring helps you understand application health, debug issues, and maintain reliability.

---

## Part 1 — Fundamentals

### What Is Cloud Monitoring?

Cloud Monitoring is GCP's fully managed observability platform that collects metrics from infrastructure, applications, and services—then lets you visualize, alert, and analyze that data to maintain reliability.

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD MONITORING OVERVIEW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐    │
│  │ GCP Services│  │ Applications│  │  External Sources    │    │
│  │ (auto)      │  │ (custom)    │  │  (AWS, on-prem)     │    │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘    │
│         │                 │                     │                │
│         └─────────────────┼─────────────────────┘                │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Cloud Monitoring                            │    │
│  │                                                         │    │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────┐   │    │
│  │  │ Metrics  │  │Dashboards │  │    Alerting       │   │    │
│  │  │ Store    │  │ & Charts  │  │    Policies       │   │    │
│  │  └──────────┘  └───────────┘  └──────────────────┘   │    │
│  │                                                         │    │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────┐   │    │
│  │  │ Uptime   │  │  Groups   │  │ Service           │   │    │
│  │  │ Checks   │  │           │  │ Monitoring (SLOs) │   │    │
│  │  └──────────┘  └───────────┘  └──────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cloud Monitoring Capabilities

| Capability | Description |
|------------|-------------|
| **Metrics collection** | Auto-collects 1,500+ metrics from GCP services |
| **Dashboards** | Pre-built and custom dashboards with rich visualizations |
| **Alerting** | Threshold, absence, MQL-based, and budget alerts |
| **Uptime checks** | HTTP(S), TCP, and ICMP probes from global locations |
| **Custom metrics** | Write your own application/business metrics |
| **SLO monitoring** | Define and track Service Level Objectives |
| **Groups** | Dynamically organize resources for monitoring |
| **MQL** | Powerful query language for complex metric analysis |
| **Multi-project** | Metrics scopes aggregate across projects |

### Cross-Cloud Comparison

| Feature | GCP Cloud Monitoring | AWS CloudWatch | Azure Monitor |
|---------|---------------------|----------------|---------------|
| Service | Cloud Monitoring | CloudWatch | Azure Monitor |
| Metrics | Auto (1,500+ types) | Auto (namespace-based) | Auto (platform metrics) |
| Custom metrics | Custom metric descriptors | PutMetricData | Custom metrics |
| Dashboards | Built-in + custom JSON | CloudWatch Dashboards | Workbooks + Dashboards |
| Alerting | Alerting policies | Alarms | Alert rules |
| Uptime/Synthetic | Uptime checks + Synthetic | Synthetics | Availability tests |
| Query language | MQL + PromQL | Metrics Insights | KQL (Kusto) |
| SLOs | Service Monitoring | SLI/SLO via CloudWatch | SLA tracking |
| APM | Cloud Trace + Profiler | X-Ray | Application Insights |
| Multi-account | Metrics scopes | Cross-account observability | Azure Lighthouse |
| Prometheus | Managed Prometheus | AMP (Managed Prometheus) | Managed Prometheus |

### Pricing Overview

| Component | Cost |
|-----------|------|
| GCP metrics (first 150M chargeable API calls) | Free |
| Custom/external metrics ingestion | ~$0.30 per 1,000 samples (150–100B) |
| Metrics storage | Included (6 weeks at full resolution) |
| Alerting policies | First 500 free, then ~$1.50/month each |
| Uptime checks | First 100 free per metrics scope |
| Notification channels | Free |
| Monitoring API calls | First 1M free/month |
| Dashboards | Free |

> **Note**: GCP built-in metrics (compute, GKE, Cloud SQL, etc.) are collected and stored for free. You only pay for custom metrics, external metrics, or very high volumes.

---

## Part 2 — Architecture & Core Concepts

### Metrics Scopes

```
┌──────────────────────────────────────────────────────────────────┐
│                    METRICS SCOPES                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  A Metrics Scope determines which projects' metrics you can see:  │
│                                                                    │
│  ┌──────────────────────────────────────────────────┐            │
│  │  Scoping Project (hosts the metrics scope)        │            │
│  │  Project: monitoring-hub                          │            │
│  │                                                    │            │
│  │  Monitored Projects:                              │            │
│  │  ├── project-a (prod)    ← metrics visible       │            │
│  │  ├── project-b (staging) ← metrics visible       │            │
│  │  ├── project-c (dev)     ← metrics visible       │            │
│  │  └── aws-connector       ← AWS metrics visible   │            │
│  │                                                    │            │
│  │  Dashboards, alerts, and uptime checks in the     │            │
│  │  scoping project can reference ALL monitored      │            │
│  │  projects' metrics.                               │            │
│  └──────────────────────────────────────────────────┘            │
│                                                                    │
│  Key rules:                                                        │
│  • Every project is its own scoping project by default            │
│  • You can add up to 375 monitored projects per scope            │
│  • A project can be monitored by multiple scopes                 │
│  • AWS accounts connected via connector projects                 │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Metric type** | What is measured (e.g., `compute.googleapis.com/instance/cpu/utilization`) |
| **Time series** | Stream of timestamped metric data points for a resource |
| **Resource type** | The monitored entity (e.g., `gce_instance`, `gke_container`) |
| **Labels** | Key-value metadata that identify specific time series |
| **Metric kind** | GAUGE (point-in-time), DELTA (change), CUMULATIVE (running total) |
| **Value type** | INT64, DOUBLE, BOOL, STRING, DISTRIBUTION |
| **Alignment** | Aggregating data points over a time window |
| **Reduction** | Combining multiple time series into fewer |

### Metric Types Structure

```
┌──────────────────────────────────────────────────────────────┐
│                  METRIC TYPE ANATOMY                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  compute.googleapis.com/instance/cpu/utilization              │
│  ├── Domain: compute.googleapis.com                           │
│  ├── Resource: instance                                       │
│  └── Metric: cpu/utilization                                  │
│                                                                │
│  Categories:                                                   │
│  ┌────────────────────────────────────────────────────┐      │
│  │ GCP Metrics (auto-collected)                        │      │
│  │ • compute.googleapis.com/...                        │      │
│  │ • container.googleapis.com/...                      │      │
│  │ • cloudsql.googleapis.com/...                       │      │
│  │ • loadbalancing.googleapis.com/...                  │      │
│  │ • run.googleapis.com/...                            │      │
│  └────────────────────────────────────────────────────┘      │
│  ┌────────────────────────────────────────────────────┐      │
│  │ Custom Metrics (user-defined)                       │      │
│  │ • custom.googleapis.com/my_app/requests             │      │
│  └────────────────────────────────────────────────────┘      │
│  ┌────────────────────────────────────────────────────┐      │
│  │ External Metrics (from outside GCP)                 │      │
│  │ • external.googleapis.com/prometheus/...            │      │
│  │ • external.googleapis.com/aws/...                   │      │
│  └────────────────────────────────────────────────────┘      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Metric Kinds

| Kind | Description | Example |
|------|-------------|---------|
| **GAUGE** | Value at a point in time | CPU utilization (currently 75%) |
| **DELTA** | Change since last data point | Request count in last 60s |
| **CUMULATIVE** | Running total since start | Total bytes sent since boot |

### Data Retention

| Resolution | Retention |
|-----------|-----------|
| Full resolution (raw) | 6 weeks |
| 10-minute aligned | 6 months |
| 1-hour aligned | 2 years (for predefined metrics) |
| Custom metrics (aligned) | 2 years |

---

## Part 3 — Metrics & Time Series

### Common GCP Metrics

| Service | Metric | Description |
|---------|--------|-------------|
| Compute Engine | `instance/cpu/utilization` | CPU usage (0.0–1.0) |
| Compute Engine | `instance/disk/read_bytes_count` | Disk read bytes |
| Compute Engine | `instance/network/sent_bytes_count` | Network egress |
| GKE | `container/cpu/usage_time` | Container CPU seconds |
| GKE | `container/memory/used_bytes` | Container memory |
| GKE | `container/restart_count` | Pod restart count |
| Cloud SQL | `database/cpu/utilization` | DB CPU usage |
| Cloud SQL | `database/disk/utilization` | DB disk usage |
| Cloud SQL | `database/postgresql/num_backends` | Active connections |
| Cloud Run | `request_count` | Requests received |
| Cloud Run | `request_latencies` | Request latency distribution |
| Cloud Run | `container/instance_count` | Running instances |
| Load Balancer | `https/request_count` | LB request count |
| Load Balancer | `https/total_latencies` | LB latency |
| Pub/Sub | `subscription/num_undelivered_messages` | Backlog size |
| Cloud Functions | `function/execution_count` | Invocations |
| Cloud Functions | `function/execution_times` | Duration |

### Reading Metrics with Monitoring API

```python
from google.cloud import monitoring_v3
from google.protobuf import timestamp_pb2
import time

client = monitoring_v3.MetricServiceClient()
project_name = f"projects/my-project"

# Define time interval (last 10 minutes)
now = time.time()
interval = monitoring_v3.TimeInterval(
    end_time={"seconds": int(now)},
    start_time={"seconds": int(now - 600)},
)

# Query CPU utilization for all instances
results = client.list_time_series(
    request={
        "name": project_name,
        "filter": 'metric.type = "compute.googleapis.com/instance/cpu/utilization"',
        "interval": interval,
        "view": monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
    }
)

for series in results:
    instance = series.resource.labels["instance_id"]
    for point in series.points:
        print(f"Instance {instance}: CPU = {point.value.double_value:.2%}")
```

### Filtering Time Series

```
# Filter syntax for Monitoring API and MQL
metric.type = "compute.googleapis.com/instance/cpu/utilization"
resource.type = "gce_instance"
resource.labels.zone = "us-central1-a"
metric.labels.instance_name = starts_with("web-")
```

### Alignment and Aggregation

```
┌──────────────────────────────────────────────────────────────┐
│            ALIGNMENT & AGGREGATION                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Raw data points (every 60s):                                 │
│  •  •  •  •  •  •  •  •  •  •  •  •  (12 points)           │
│                                                                │
│  After ALIGNMENT (5-min window, ALIGN_MEAN):                  │
│  ├─────────────┤├─────────────┤├──────────────┤              │
│       0.45           0.62           0.58                       │
│                                                                │
│  After REDUCTION (across instances, REDUCE_MEAN):             │
│  Multiple instances → single value per window                 │
│  Instance A: 0.45, 0.62, 0.58                                │
│  Instance B: 0.30, 0.55, 0.42                                │
│  Instance C: 0.60, 0.70, 0.65                                │
│  REDUCE_MEAN: 0.45, 0.62, 0.55                               │
│                                                                │
│  Aligners:                                                     │
│  ALIGN_MEAN, ALIGN_MAX, ALIGN_MIN, ALIGN_SUM,                │
│  ALIGN_COUNT, ALIGN_RATE, ALIGN_PERCENTILE_99                 │
│                                                                │
│  Reducers:                                                     │
│  REDUCE_MEAN, REDUCE_MAX, REDUCE_MIN, REDUCE_SUM,            │
│  REDUCE_COUNT, REDUCE_PERCENTILE_99                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 4 — Metrics Explorer

### Using Metrics Explorer

Metrics Explorer is the primary tool for ad-hoc metric exploration in the Console:

```
┌──────────────────────────────────────────────────────────────┐
│              METRICS EXPLORER                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Console → Monitoring → Metrics Explorer                      │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Resource type: [VM Instance           ▼]            │    │
│  │  Metric:        [CPU utilization       ▼]            │    │
│  │  Filter:        [zone = us-central1-a    ]           │    │
│  │  Group by:      [instance_name           ]           │    │
│  │  Aggregator:    [mean                  ▼]            │    │
│  │  Alignment:     [1 minute              ▼]            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Chart Area                                           │    │
│  │    ╭──────────────────────────────────╮              │    │
│  │  1 │       ╱╲    ╱╲                   │              │    │
│  │    │      ╱  ╲  ╱  ╲     ╱╲          │              │    │
│  │0.5 │─────╱────╲╱────╲───╱──╲─────────│              │    │
│  │    │    ╱              ╲╱    ╲        │              │    │
│  │  0 │───────────────────────────────────│              │    │
│  │    ╰──────────────────────────────────╯              │    │
│  │     12:00   12:15   12:30   12:45   13:00            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Actions: [Save to Dashboard] [Create Alert] [View as MQL]   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Metrics Explorer Features

| Feature | Description |
|---------|-------------|
| Resource & metric picker | Browse available metrics by service |
| Filters | Narrow by labels (zone, instance, etc.) |
| Group by | Split chart by a label dimension |
| Aggregation | Choose aligner + reducer |
| Time range | 1h, 6h, 12h, 1d, 7d, 30d, custom |
| Chart types | Line, stacked area, bar, heatmap |
| PromQL mode | Write PromQL queries directly |
| MQL mode | Write MQL queries directly |
| Save to dashboard | Add chart to a new/existing dashboard |
| Create alert | Directly create alert from the query |

### PromQL in Metrics Explorer

```promql
# CPU utilization above 80% for web instances
compute_googleapis_com:instance_cpu_utilization{
  instance_name=~"web-.*"
} > 0.8

# 99th percentile request latency for Cloud Run
rate(run_googleapis_com:request_latencies_count[5m])

# GKE container memory usage by namespace
sum by (namespace_name) (
  container_googleapis_com:container_memory_used_bytes{
    cluster_name="prod-cluster"
  }
)
```

---

## Part 5 — Dashboards

### Dashboard Types

| Type | Description |
|------|-------------|
| **Predefined** | Auto-generated for GCP services (Compute, GKE, Cloud SQL) |
| **Custom** | User-created with any combination of widgets |
| **JSON-based** | Defined as JSON for version control and automation |

### Creating Custom Dashboards

```
┌──────────────────────────────────────────────────────────────┐
│              CUSTOM DASHBOARD                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Dashboard: "Production Overview"                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │    │
│  │  │ CPU Usage   │  │ Memory      │  │ Disk I/O    │ │    │
│  │  │ (line chart)│  │ (line chart)│  │ (bar chart) │ │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │    │
│  │                                                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │    │
│  │  │ Request     │  │ Error Rate  │  │ Latency     │ │    │
│  │  │ Count       │  │ (scorecard) │  │ (p99)       │ │    │
│  │  │ (scorecard) │  │             │  │ (heatmap)   │ │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │    │
│  │                                                      │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │ Alert Summary (incidents table)                │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Widget types:                                                 │
│  • Line chart        • Stacked area     • Bar chart           │
│  • Scorecard         • Gauge            • Table               │
│  • Heatmap          • Log panel         • Alert chart         │
│  • Text (markdown)   • Blank spacer     • Incident list       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Dashboard JSON Definition

```json
{
  "displayName": "Production Overview",
  "mosaicLayout": {
    "columns": 12,
    "tiles": [
      {
        "xPos": 0,
        "yPos": 0,
        "width": 4,
        "height": 4,
        "widget": {
          "title": "CPU Utilization",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" resource.type=\"gce_instance\"",
                    "aggregation": {
                      "alignmentPeriod": "60s",
                      "perSeriesAligner": "ALIGN_MEAN",
                      "crossSeriesReducer": "REDUCE_MEAN",
                      "groupByFields": ["resource.labels.instance_id"]
                    }
                  }
                },
                "plotType": "LINE"
              }
            ]
          }
        }
      },
      {
        "xPos": 4,
        "yPos": 0,
        "width": 4,
        "height": 4,
        "widget": {
          "title": "Error Rate",
          "scorecard": {
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"run.googleapis.com/request_count\" resource.type=\"cloud_run_revision\" metric.labels.response_code_class=\"5xx\"",
                "aggregation": {
                  "alignmentPeriod": "300s",
                  "perSeriesAligner": "ALIGN_RATE"
                }
              }
            },
            "thresholds": [
              {"value": 0.01, "color": "YELLOW", "direction": "ABOVE"},
              {"value": 0.05, "color": "RED", "direction": "ABOVE"}
            ]
          }
        }
      }
    ]
  }
}
```

### Managing Dashboards via CLI

```bash
# Create dashboard from JSON
gcloud monitoring dashboards create --config-from-file=dashboard.json

# List dashboards
gcloud monitoring dashboards list

# Describe a dashboard
gcloud monitoring dashboards describe DASHBOARD_ID

# Update dashboard
gcloud monitoring dashboards update DASHBOARD_ID \
    --config-from-file=updated-dashboard.json

# Delete dashboard
gcloud monitoring dashboards delete DASHBOARD_ID
```

---

## Part 6 — Alerting Policies

### Alerting Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                ALERTING ARCHITECTURE                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────────────────────────────┐              │
│  │           Alerting Policy                   │              │
│  │                                            │              │
│  │  ┌────────────────────────────────────┐   │              │
│  │  │  Condition(s)                       │   │              │
│  │  │  • Metric threshold exceeded        │   │              │
│  │  │  • Metric absent                    │   │              │
│  │  │  • MQL-based condition              │   │              │
│  │  │  • Log-based condition              │   │              │
│  │  │  • Process health condition         │   │              │
│  │  └──────────────────┬─────────────────┘   │              │
│  │                     │ triggers             │              │
│  │                     ▼                      │              │
│  │  ┌────────────────────────────────────┐   │              │
│  │  │  Notification Channel(s)            │   │              │
│  │  │  • Email                            │   │              │
│  │  │  • Slack                            │   │              │
│  │  │  • PagerDuty                        │   │              │
│  │  │  • Webhook                          │   │              │
│  │  │  • SMS                              │   │              │
│  │  │  • Pub/Sub                          │   │              │
│  │  └────────────────────────────────────┘   │              │
│  │                                            │              │
│  │  Configuration:                            │              │
│  │  • Duration: how long condition must hold  │              │
│  │  • Combiner: AND/OR for multiple conditions│              │
│  │  • Documentation: runbook, context         │              │
│  │  • Auto-close: when to resolve incident    │              │
│  └────────────────────────────────────────────┘              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Condition Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Metric threshold** | Value above/below threshold for duration | CPU > 80% for 5 min |
| **Metric absence** | No data received for duration | Heartbeat missing |
| **MQL condition** | Complex query using MQL | Ratio of errors to requests |
| **Log-based** | Log entry matches filter | Error log pattern detected |
| **Forecast** | Predicted to violate in future | Disk full in 24 hours |
| **Process health** | Process not running | nginx not running |

### Creating Alerting Policies

```bash
# Create alert via CLI (from JSON)
gcloud alpha monitoring policies create --policy-from-file=alert-policy.json

# List policies
gcloud alpha monitoring policies list

# Describe policy
gcloud alpha monitoring policies describe POLICY_ID

# Delete policy
gcloud alpha monitoring policies delete POLICY_ID

# Enable/disable policy
gcloud alpha monitoring policies update POLICY_ID --enabled
gcloud alpha monitoring policies update POLICY_ID --no-enabled
```

### Alert Policy JSON Example

```json
{
  "displayName": "High CPU Utilization",
  "documentation": {
    "content": "CPU utilization has exceeded 80% for 5 minutes.\n\n**Runbook**: Check for runaway processes, consider scaling.",
    "mimeType": "text/markdown"
  },
  "conditions": [
    {
      "displayName": "CPU above 80%",
      "conditionThreshold": {
        "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" resource.type=\"gce_instance\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.8,
        "duration": "300s",
        "aggregations": [
          {
            "alignmentPeriod": "60s",
            "perSeriesAligner": "ALIGN_MEAN"
          }
        ],
        "trigger": {
          "count": 1
        }
      }
    }
  ],
  "combiner": "OR",
  "notificationChannels": [
    "projects/my-project/notificationChannels/CHANNEL_ID"
  ],
  "alertStrategy": {
    "autoClose": "1800s"
  }
}
```

### Multiple Conditions (AND/OR)

```json
{
  "displayName": "High CPU AND Memory",
  "conditions": [
    {
      "displayName": "CPU > 80%",
      "conditionThreshold": {
        "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.8,
        "duration": "300s"
      }
    },
    {
      "displayName": "Memory > 90%",
      "conditionThreshold": {
        "filter": "metric.type=\"agent.googleapis.com/memory/percent_used\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 90,
        "duration": "300s"
      }
    }
  ],
  "combiner": "AND"
}
```

### Forecast-Based Alert

```json
{
  "displayName": "Disk Full Forecast",
  "conditions": [
    {
      "displayName": "Disk predicted full in 24h",
      "conditionThreshold": {
        "filter": "metric.type=\"compute.googleapis.com/instance/disk/utilization\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.95,
        "duration": "0s",
        "forecastOptions": {
          "forecastHorizon": "86400s"
        }
      }
    }
  ]
}
```

---

## Part 7 — Notification Channels

### Available Channel Types

| Type | Description | Configuration |
|------|-------------|---------------|
| Email | Email notification | Email address |
| Slack | Slack channel message | Workspace + channel |
| PagerDuty | PagerDuty incident | Service key |
| Webhooks | HTTP POST to URL | Endpoint URL |
| SMS | Text message | Phone number |
| Pub/Sub | Publish to topic | Topic name |
| Google Chat | Google Chat space | Webhook URL |
| Mobile app | Cloud Console mobile | Auto (install app) |

### Managing Notification Channels

```bash
# Create email channel
gcloud alpha monitoring channels create \
    --display-name="Ops Team Email" \
    --type=email \
    --channel-labels=email_address=ops-team@example.com

# Create Slack channel
gcloud alpha monitoring channels create \
    --display-name="Alerts Slack" \
    --type=slack \
    --channel-labels=channel_name="#alerts"

# Create PagerDuty channel
gcloud alpha monitoring channels create \
    --display-name="PagerDuty On-Call" \
    --type=pagerduty \
    --channel-labels=service_key=YOUR_INTEGRATION_KEY

# Create webhook channel
gcloud alpha monitoring channels create \
    --display-name="Custom Webhook" \
    --type=webhook_tokenauth \
    --channel-labels=url="https://my-webhook.example.com/alerts"

# Create Pub/Sub channel
gcloud alpha monitoring channels create \
    --display-name="Alert Pub/Sub" \
    --type=pubsub \
    --channel-labels=topic="projects/my-project/topics/monitoring-alerts"

# List channels
gcloud alpha monitoring channels list

# Delete channel
gcloud alpha monitoring channels delete CHANNEL_ID
```

### Notification Channel Verification

```bash
# Some channels require verification (email, SMS)
gcloud alpha monitoring channels verify CHANNEL_ID \
    --verification-code=CODE
```

### Alert Notification JSON Structure (Pub/Sub / Webhook)

```json
{
  "incident": {
    "incident_id": "0.abc123",
    "scoping_project_id": "my-project",
    "scoping_project_number": 123456789,
    "url": "https://console.cloud.google.com/monitoring/alerting/incidents/...",
    "started_at": 1632150000,
    "ended_at": null,
    "state": "open",
    "resource_id": "1234567890",
    "resource_name": "web-server-1",
    "resource_display_name": "web-server-1",
    "resource_type_display_name": "VM Instance",
    "resource": {
      "type": "gce_instance",
      "labels": {
        "instance_id": "1234567890",
        "zone": "us-central1-a",
        "project_id": "my-project"
      }
    },
    "metric": {
      "type": "compute.googleapis.com/instance/cpu/utilization",
      "displayName": "CPU utilization"
    },
    "policy_name": "High CPU Utilization",
    "condition_name": "CPU above 80%",
    "threshold_value": "0.8",
    "observed_value": "0.92",
    "summary": "CPU utilization for web-server-1 is above the threshold of 80%"
  }
}
```

---

## Part 8 — Uptime Checks

### What Are Uptime Checks?

Uptime checks probe your services from multiple global locations at regular intervals to verify availability:

```
┌──────────────────────────────────────────────────────────────┐
│                 UPTIME CHECKS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Global probe locations:                                       │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐         │
│  │Oregon│  │Iowa  │  │São   │  │Belgium│  │Singa-│         │
│  │      │  │      │  │Paulo │  │       │  │pore  │         │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘         │
│     │         │         │         │         │               │
│     └─────────┴─────────┴────┬────┴─────────┘               │
│                               │                               │
│                               ▼                               │
│                    ┌──────────────────┐                       │
│                    │  Your Service    │                       │
│                    │  (HTTP/TCP/ICMP) │                       │
│                    └──────────────────┘                       │
│                                                                │
│  Check passes if:                                              │
│  • HTTP: Response code matches (e.g., 2xx)                    │
│  • HTTP: Response body contains expected content              │
│  • TCP: Connection established                                │
│  • ICMP: Ping responds                                        │
│                                                                │
│  Check fails → creates incident (triggers alert)              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Uptime Check Types

| Type | Protocol | Use Case |
|------|----------|----------|
| HTTP(S) | HTTP/HTTPS | Web apps, APIs, health endpoints |
| TCP | TCP | Database ports, custom services |
| ICMP | ICMP Ping | Network connectivity |

### Creating Uptime Checks

```bash
# Create HTTP uptime check
gcloud monitoring uptime create "Production API" \
    --resource-type=uptime-url \
    --hostname="api.example.com" \
    --path="/health" \
    --protocol=HTTPS \
    --check-interval=60s \
    --timeout=10s \
    --content-match-content="ok" \
    --regions=USA,EUROPE,ASIA_PACIFIC

# Create TCP uptime check
gcloud monitoring uptime create "Database Port" \
    --resource-type=uptime-url \
    --hostname="db.internal.example.com" \
    --protocol=TCP \
    --port=5432 \
    --check-interval=300s

# List uptime checks
gcloud monitoring uptime list-configs

# Delete uptime check
gcloud monitoring uptime delete CHECK_ID
```

### Uptime Check with Authentication

```json
{
  "displayName": "Authenticated API Check",
  "monitoredResource": {
    "type": "uptime_url",
    "labels": {
      "host": "api.example.com",
      "project_id": "my-project"
    }
  },
  "httpCheck": {
    "path": "/api/v1/health",
    "port": 443,
    "useSsl": true,
    "validateSsl": true,
    "requestMethod": "GET",
    "authInfo": {
      "username": "monitor",
      "password": "monitor-password"
    },
    "headers": {
      "X-Custom-Header": "monitoring"
    },
    "acceptedResponseStatusCodes": [
      {"statusClass": "STATUS_CLASS_2XX"}
    ],
    "contentMatchers": [
      {
        "content": "\"status\":\"healthy\"",
        "matcher": "CONTAINS_STRING"
      }
    ]
  },
  "period": "60s",
  "timeout": "10s",
  "selectedRegions": ["USA", "EUROPE", "ASIA_PACIFIC"]
}
```

### Synthetic Monitors (Advanced Uptime)

```bash
# Synthetic monitors use Cloud Functions to run custom checks
# More complex than basic uptime — can test multi-step workflows

# Example: Cloud Function for synthetic monitoring
# Deployed as a Cloud Function, called by Monitoring
```

```python
# Synthetic monitor function
import functions_framework
import requests

@functions_framework.http
def synthetic_check(request):
    """Multi-step synthetic check."""
    # Step 1: Check login endpoint
    login_resp = requests.post(
        "https://app.example.com/api/login",
        json={"user": "test", "pass": "test123"},
        timeout=5
    )
    assert login_resp.status_code == 200
    token = login_resp.json()["token"]
    
    # Step 2: Check protected endpoint
    data_resp = requests.get(
        "https://app.example.com/api/data",
        headers={"Authorization": f"Bearer {token}"},
        timeout=5
    )
    assert data_resp.status_code == 200
    assert len(data_resp.json()["items"]) > 0
    
    return "OK", 200
```

---

## Part 9 — Groups

### What Are Monitoring Groups?

Groups let you dynamically organize resources for monitoring. Instead of manually selecting specific instances, you define criteria—and resources that match are automatically included.

```
┌──────────────────────────────────────────────────────────────┐
│                  MONITORING GROUPS                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Group: "Production Web Servers"                              │
│  Filter: resource.metadata.name = starts_with("prod-web")    │
│                                                                │
│  Members (auto-populated):                                    │
│  ├── prod-web-001                                             │
│  ├── prod-web-002                                             │
│  ├── prod-web-003                                             │
│  └── prod-web-004  (new instance auto-added!)                │
│                                                                │
│  Nested Groups (hierarchy):                                   │
│  ┌────────────────────────────────────────────┐              │
│  │  All Production                            │              │
│  │  ├── Web Tier                              │              │
│  │  │   ├── prod-web-001                      │              │
│  │  │   └── prod-web-002                      │              │
│  │  ├── App Tier                              │              │
│  │  │   ├── prod-app-001                      │              │
│  │  │   └── prod-app-002                      │              │
│  │  └── Data Tier                             │              │
│  │      ├── prod-db-001                       │              │
│  │      └── prod-cache-001                    │              │
│  └────────────────────────────────────────────┘              │
│                                                                │
│  Use in: Dashboards, alerts, uptime checks                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Group Membership Criteria

| Criteria | Example |
|----------|---------|
| Name prefix | `resource.metadata.name = starts_with("web-")` |
| Labels | `resource.metadata.user_labels.env = "prod"` |
| Region | `resource.metadata.region = "us-central1"` |
| Tags | `resource.metadata.tags = "web-server"` |
| Cloud labels | `metadata.system_labels.cloud = "gcp"` |
| Combined | Multiple criteria with AND/OR logic |

### Creating Groups

```bash
# Create a monitoring group
gcloud alpha monitoring groups create \
    --display-name="Production Web Servers" \
    --filter='resource.metadata.user_labels.env="prod" AND resource.metadata.user_labels.tier="web"'

# Create child group (nested)
gcloud alpha monitoring groups create \
    --display-name="Web US Central" \
    --parent-name=PARENT_GROUP_NAME \
    --filter='resource.metadata.region="us-central1"'

# List groups
gcloud alpha monitoring groups list

# List members of a group
gcloud alpha monitoring groups members list --group-name=GROUP_NAME
```

---

## Part 10 — Custom Metrics

### Writing Custom Metrics

```
┌──────────────────────────────────────────────────────────────┐
│               CUSTOM METRICS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Use custom metrics for:                                       │
│  • Application-level metrics (queue depth, active users)      │
│  • Business metrics (orders/minute, revenue)                  │
│  • Metrics not captured by GCP auto-collection                │
│                                                                │
│  Writing methods:                                              │
│  1. Monitoring API (direct write)                             │
│  2. OpenTelemetry SDK                                         │
│  3. Prometheus client (via Managed Prometheus)                │
│  4. Ops Agent (custom StatsD/collectd)                        │
│                                                                │
│  Metric type format:                                           │
│  custom.googleapis.com/my_app/active_users                    │
│  ├── Domain: custom.googleapis.com                            │
│  └── Path: my_app/active_users                                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating a Custom Metric Descriptor

```python
from google.cloud import monitoring_v3

client = monitoring_v3.MetricServiceClient()
project_name = f"projects/my-project"

descriptor = monitoring_v3.MetricDescriptor(
    type="custom.googleapis.com/my_app/active_users",
    metric_kind=monitoring_v3.MetricDescriptor.MetricKind.GAUGE,
    value_type=monitoring_v3.MetricDescriptor.ValueType.INT64,
    description="Number of currently active users",
    display_name="Active Users",
    unit="1",  # dimensionless
    labels=[
        monitoring_v3.LabelDescriptor(
            key="region",
            value_type=monitoring_v3.LabelDescriptor.ValueType.STRING,
            description="Application region",
        ),
    ],
)

created = client.create_metric_descriptor(
    name=project_name, metric_descriptor=descriptor
)
print(f"Created: {created.name}")
```

### Writing Custom Metric Data Points

```python
from google.cloud import monitoring_v3
from google.protobuf import timestamp_pb2
import time

client = monitoring_v3.MetricServiceClient()
project_name = f"projects/my-project"

# Create a time series data point
series = monitoring_v3.TimeSeries()
series.metric.type = "custom.googleapis.com/my_app/active_users"
series.metric.labels["region"] = "us-central1"

series.resource.type = "global"
series.resource.labels["project_id"] = "my-project"

# Write current value
now = time.time()
point = monitoring_v3.Point()
point.value.int64_value = 1523
point.interval.end_time = {"seconds": int(now)}

series.points = [point]

client.create_time_series(
    request={
        "name": project_name,
        "time_series": [series],
    }
)
print("Written: active_users = 1523")
```

### Writing Custom Metrics with OpenTelemetry

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.cloud_monitoring import CloudMonitoringMetricsExporter
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

# Setup exporter
exporter = CloudMonitoringMetricsExporter(project_id="my-project")
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=60000)
provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(provider)

meter = metrics.get_meter("my_app")

# Create metrics
active_users = meter.create_up_down_counter(
    name="active_users",
    description="Currently active users",
    unit="1",
)

request_counter = meter.create_counter(
    name="requests_total",
    description="Total HTTP requests",
    unit="1",
)

request_duration = meter.create_histogram(
    name="request_duration",
    description="Request duration in milliseconds",
    unit="ms",
)

# Use in application
active_users.add(1, {"region": "us-central1"})
request_counter.add(1, {"method": "GET", "path": "/api/users"})
request_duration.record(45.2, {"method": "GET", "path": "/api/users"})
```

### Ops Agent Custom Metrics (StatsD)

```yaml
# /etc/google-cloud-ops-agent/config.yaml
metrics:
  receivers:
    statsd:
      type: statsd
      collection_interval: 30s
      listen_address: 0.0.0.0
      listen_port: 8125
  service:
    pipelines:
      statsd_pipeline:
        receivers: [statsd]
```

```python
# Send metrics via StatsD from your app
import statsd
c = statsd.StatsClient('localhost', 8125)

c.gauge('my_app.queue_depth', 42)
c.incr('my_app.requests')
c.timing('my_app.response_time', 320)
```

---

## Part 11 — Monitoring Query Language (MQL)

### MQL Overview

MQL is a text-based query language for Cloud Monitoring that provides more power and flexibility than the graphical Metrics Explorer:

```
┌──────────────────────────────────────────────────────────────┐
│                        MQL                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Structure:                                                    │
│  fetch RESOURCE_TYPE::METRIC_TYPE                             │
│  | filter CONDITIONS                                          │
│  | align ALIGNER(PERIOD)                                      │
│  | group_by [LABELS], REDUCER                                 │
│  | every PERIOD                                                │
│  | condition EXPRESSION                                        │
│                                                                │
│  Example:                                                      │
│  fetch gce_instance::compute.googleapis.com/instance/cpu/     │
│    utilization                                                 │
│  | filter zone = 'us-central1-a'                              │
│  | group_by [instance_name], mean(val())                      │
│  | every 5m                                                    │
│  | condition val() > 0.8 '5m'                                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### MQL Examples

```sql
-- CPU utilization grouped by instance
fetch gce_instance
| metric 'compute.googleapis.com/instance/cpu/utilization'
| filter zone = 'us-central1-a'
| group_by [resource.instance_id], mean(val())
| every 5m

-- Request count rate for Cloud Run
fetch cloud_run_revision
| metric 'run.googleapis.com/request_count'
| align rate(1m)
| group_by [resource.service_name], sum(val())

-- Error ratio (5xx / total requests)
fetch cloud_run_revision
| metric 'run.googleapis.com/request_count'
| align rate(5m)
| {
    filter metric.response_code_class = '5xx'
    ;
    ident
  }
| group_by [], [sum(val())]
| ratio

-- Disk utilization forecast
fetch gce_instance
| metric 'compute.googleapis.com/instance/disk/utilization'
| group_by [resource.instance_id], mean(val())
| every 1h
| window 7d
| forecast 24h

-- Top 5 instances by CPU
fetch gce_instance
| metric 'compute.googleapis.com/instance/cpu/utilization'
| group_by [resource.instance_id], mean(val())
| every 5m
| top 5

-- Percentile (p99 latency)
fetch cloud_run_revision
| metric 'run.googleapis.com/request_latencies'
| align delta(1m)
| group_by [resource.service_name], percentile(val(), 99)
```

### MQL for Alerting Conditions

```json
{
  "displayName": "Error Rate > 5%",
  "conditions": [
    {
      "displayName": "Error ratio exceeds 5%",
      "conditionMonitoringQueryLanguage": {
        "query": "fetch cloud_run_revision | metric 'run.googleapis.com/request_count' | align rate(5m) | { filter metric.response_code_class = '5xx' ; ident } | group_by [], [sum(val())] | ratio | condition val() > 0.05 '5m'",
        "duration": "300s"
      }
    }
  ]
}
```

### MQL vs PromQL

| Feature | MQL | PromQL |
|---------|-----|--------|
| Syntax style | Pipe-based (shell-like) | Functional |
| GCP native | Yes | Via Managed Prometheus |
| Ratio queries | Built-in `ratio` | Manual division |
| Forecast | Built-in | Not native |
| Learning curve | Moderate | Moderate |
| External tooling | GCP Console only | Grafana, many tools |

---

## Part 12 — Service Monitoring & SLOs

### What Is Service Monitoring?

Service Monitoring lets you define Services and set Service Level Objectives (SLOs) to track reliability:

```
┌──────────────────────────────────────────────────────────────┐
│              SERVICE MONITORING & SLOs                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────────────────────────┐                  │
│  │  Service: "checkout-api"               │                  │
│  │                                        │                  │
│  │  SLI (indicator): Request success rate │                  │
│  │  SLO (objective): 99.9% over 30 days  │                  │
│  │                                        │                  │
│  │  Error Budget:                         │                  │
│  │  ┌──────────────────────────────────┐ │                  │
│  │  │ ████████████████████████░░░░░░░░ │ │                  │
│  │  │ 72% remaining (21.6 min left)   │ │                  │
│  │  └──────────────────────────────────┘ │                  │
│  │                                        │                  │
│  │  Burn rate alert:                      │                  │
│  │  • Fast burn: 14.4x → Page (1h)       │                  │
│  │  • Slow burn: 6x → Ticket (6h)        │                  │
│  └────────────────────────────────────────┘                  │
│                                                                │
│  Supported service types:                                      │
│  • Cloud Run (auto-detected)                                   │
│  • GKE services (Istio/Anthos)                                │
│  • App Engine                                                  │
│  • Cloud Endpoints                                             │
│  • Custom services                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### SLI Types

| SLI Type | Description | Example |
|----------|-------------|---------|
| **Availability** | Proportion of successful requests | 99.9% of requests return 2xx/3xx |
| **Latency** | Proportion within threshold | 95% of requests < 200ms |
| **Quality** | Proportion of "good" responses | 99% of responses < 1MB |

### Creating a Service and SLO

```bash
# Create a custom service
gcloud monitoring services create \
    --display-name="Checkout API" \
    --service-id=checkout-api

# Create availability SLO (99.9% over 30 days)
gcloud monitoring slos create \
    --service=checkout-api \
    --display-name="Availability SLO" \
    --goal=0.999 \
    --rolling-period=30d \
    --request-based-sli \
    --good-total-ratio-filter='metric.type="run.googleapis.com/request_count" resource.type="cloud_run_revision" metric.labels.response_code_class!="5xx"' \
    --total-ratio-filter='metric.type="run.googleapis.com/request_count" resource.type="cloud_run_revision"'

# Create latency SLO (95% under 200ms)
gcloud monitoring slos create \
    --service=checkout-api \
    --display-name="Latency SLO" \
    --goal=0.95 \
    --rolling-period=30d \
    --request-based-sli \
    --distribution-cut-range-min=0 \
    --distribution-cut-range-max=0.2 \
    --distribution-filter='metric.type="run.googleapis.com/request_latencies" resource.type="cloud_run_revision"'
```

### Burn Rate Alerts

```json
{
  "displayName": "SLO Burn Rate - Fast",
  "conditions": [
    {
      "displayName": "Burn rate > 14.4x (page)",
      "conditionThreshold": {
        "filter": "select_slo_burn_rate(\"projects/my-project/services/checkout-api/serviceLevelObjectives/SLO_ID\", \"60m\")",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 14.4,
        "duration": "60s"
      }
    }
  ]
}
```

---

## Part 13 — Integration with Other Services

### Cloud Monitoring Ecosystem

```
┌──────────────────────────────────────────────────────────────┐
│          MONITORING ECOSYSTEM INTEGRATION                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────────────────────────────────────────┐     │
│  │                 Cloud Monitoring                      │     │
│  └───────────────────────┬─────────────────────────────┘     │
│                          │                                    │
│  ┌───────────┬───────────┼───────────┬───────────────┐      │
│  │           │           │           │               │      │
│  ▼           ▼           ▼           ▼               ▼      │
│ Cloud      Cloud       Cloud      Error          Managed    │
│ Logging    Trace       Profiler   Reporting     Prometheus  │
│                                                              │
│ Additional integrations:                                     │
│ • Ops Agent (system metrics, logs, app metrics)             │
│ • Managed Prometheus (PromQL, Grafana)                      │
│ • OpenTelemetry Collector                                   │
│ • Third-party: Datadog, Splunk, PagerDuty, Grafana Cloud   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Managed Service for Prometheus

```yaml
# GKE — enable Managed Prometheus
# It's enabled by default on GKE Autopilot and Standard (1.27+)

# PodMonitoring CRD to scrape your app's metrics
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: my-app-metrics
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Ops Agent Configuration

```yaml
# /etc/google-cloud-ops-agent/config.yaml
logging:
  receivers:
    app_logs:
      type: files
      include_paths:
        - /var/log/my-app/*.log
      record_log_name: my-app
  service:
    pipelines:
      app_pipeline:
        receivers: [app_logs]

metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
    nginx:
      type: nginx
      stub_status_url: http://localhost/nginx_status
  service:
    pipelines:
      default_pipeline:
        receivers: [hostmetrics, nginx]
```

### Log-Based Metrics

```bash
# Create a log-based metric (counter)
gcloud logging metrics create error_count \
    --description="Count of error log entries" \
    --log-filter='severity="ERROR" resource.type="cloud_run_revision"'

# Create distribution metric (latency from logs)
gcloud logging metrics create response_latency \
    --description="Response latency from access logs" \
    --log-filter='resource.type="cloud_run_revision" httpRequest.latency!=""' \
    --bucket-name=response_latency_dist
```

### Grafana Integration

```bash
# Use Managed Prometheus as Grafana data source
# Grafana connects to Cloud Monitoring via:
# 1. Managed Prometheus endpoint (for PromQL)
# 2. Cloud Monitoring data source plugin (for MQL/native metrics)

# Prometheus endpoint format:
# https://monitoring.googleapis.com/v1/projects/PROJECT_ID/location/global/prometheus
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Alerting Policy

```hcl
# Enable Monitoring API
resource "google_project_service" "monitoring" {
  service = "monitoring.googleapis.com"
}

# Notification channel — Email
resource "google_monitoring_notification_channel" "email" {
  display_name = "Ops Team Email"
  type         = "email"
  project      = var.project_id

  labels = {
    email_address = "ops-team@example.com"
  }
}

# Notification channel — Slack
resource "google_monitoring_notification_channel" "slack" {
  display_name = "Alerts Slack Channel"
  type         = "slack"
  project      = var.project_id

  labels = {
    channel_name = "#production-alerts"
  }

  sensitive_labels {
    auth_token = var.slack_token
  }
}

# Notification channel — PagerDuty
resource "google_monitoring_notification_channel" "pagerduty" {
  display_name = "PagerDuty On-Call"
  type         = "pagerduty"
  project      = var.project_id

  labels = {
    service_key = var.pagerduty_key
  }
}

# Alert policy — CPU threshold
resource "google_monitoring_alert_policy" "cpu_high" {
  display_name = "High CPU Utilization"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "CPU > 80% for 5 minutes"

    condition_threshold {
      filter          = "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" AND resource.type=\"gce_instance\""
      comparison      = "COMPARISON_GT"
      threshold_value = 0.8
      duration        = "300s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }

      trigger {
        count = 1
      }
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.email.name,
    google_monitoring_notification_channel.slack.name,
  ]

  documentation {
    content   = "CPU utilization exceeded 80% for 5 minutes.\n\n**Runbook**: Check for runaway processes. Consider scaling the instance group."
    mime_type = "text/markdown"
  }

  alert_strategy {
    auto_close = "1800s"
  }
}
```

### Terraform — Metric Absence Alert

```hcl
resource "google_monitoring_alert_policy" "heartbeat_missing" {
  display_name = "Heartbeat Missing"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "No heartbeat for 5 minutes"

    condition_absent {
      filter   = "metric.type=\"custom.googleapis.com/my_app/heartbeat\" AND resource.type=\"global\""
      duration = "300s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_COUNT"
      }
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.pagerduty.name,
  ]
}
```

### Terraform — Uptime Check + Alert

```hcl
# Uptime check
resource "google_monitoring_uptime_check_config" "api" {
  display_name = "Production API Health"
  project      = var.project_id
  timeout      = "10s"
  period       = "60s"

  http_check {
    path         = "/health"
    port         = 443
    use_ssl      = true
    validate_ssl = true

    accepted_response_status_codes {
      status_class = "STATUS_CLASS_2XX"
    }

    content_matchers {
      content = "\"status\":\"ok\""
      matcher = "CONTAINS_STRING"
    }
  }

  monitored_resource {
    type = "uptime_url"
    labels = {
      project_id = var.project_id
      host       = "api.example.com"
    }
  }

  selected_regions = ["USA", "EUROPE", "ASIA_PACIFIC"]
}

# Alert on uptime check failure
resource "google_monitoring_alert_policy" "uptime_alert" {
  display_name = "API Down"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "Uptime check failed"

    condition_threshold {
      filter          = "metric.type=\"monitoring.googleapis.com/uptime_check/check_passed\" AND resource.type=\"uptime_url\" AND metric.labels.check_id=\"${google_monitoring_uptime_check_config.api.uptime_check_id}\""
      comparison      = "COMPARISON_GT"
      threshold_value = 1
      duration        = "60s"

      aggregations {
        alignment_period     = "1200s"
        per_series_aligner   = "ALIGN_NEXT_OLDER"
        cross_series_reducer = "REDUCE_COUNT_FALSE"
        group_by_fields      = ["resource.label.project_id", "resource.label.host"]
      }
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.pagerduty.name,
    google_monitoring_notification_channel.slack.name,
  ]
}
```

### Terraform — Dashboard

```hcl
resource "google_monitoring_dashboard" "production" {
  dashboard_json = jsonencode({
    displayName = "Production Overview"
    mosaicLayout = {
      columns = 12
      tiles = [
        {
          xPos   = 0
          yPos   = 0
          width  = 6
          height = 4
          widget = {
            title = "CPU Utilization"
            xyChart = {
              dataSets = [{
                timeSeriesQuery = {
                  timeSeriesFilter = {
                    filter = "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" resource.type=\"gce_instance\""
                    aggregation = {
                      alignmentPeriod    = "60s"
                      perSeriesAligner   = "ALIGN_MEAN"
                      crossSeriesReducer = "REDUCE_MEAN"
                      groupByFields      = ["resource.labels.instance_id"]
                    }
                  }
                }
                plotType = "LINE"
              }]
            }
          }
        },
        {
          xPos   = 6
          yPos   = 0
          width  = 6
          height = 4
          widget = {
            title = "Request Rate"
            xyChart = {
              dataSets = [{
                timeSeriesQuery = {
                  timeSeriesFilter = {
                    filter = "metric.type=\"run.googleapis.com/request_count\" resource.type=\"cloud_run_revision\""
                    aggregation = {
                      alignmentPeriod  = "60s"
                      perSeriesAligner = "ALIGN_RATE"
                    }
                  }
                }
                plotType = "STACKED_BAR"
              }]
            }
          }
        }
      ]
    }
  })
  project = var.project_id
}
```

### Terraform — SLO

```hcl
# Custom service
resource "google_monitoring_custom_service" "api" {
  service_id   = "checkout-api"
  display_name = "Checkout API"
  project      = var.project_id
}

# Availability SLO
resource "google_monitoring_slo" "availability" {
  service      = google_monitoring_custom_service.api.service_id
  slo_id       = "availability-slo"
  display_name = "99.9% Availability"
  project      = var.project_id

  goal                = 0.999
  rolling_period_days = 30

  request_based_sli {
    good_total_ratio {
      good_service_filter = "metric.type=\"run.googleapis.com/request_count\" resource.type=\"cloud_run_revision\" metric.labels.response_code_class!=\"5xx\""
      total_service_filter = "metric.type=\"run.googleapis.com/request_count\" resource.type=\"cloud_run_revision\""
    }
  }
}
```

### gcloud CLI Reference

```bash
# ─── Metrics ─────────────────────────────────────────────────
# List metric descriptors
gcloud monitoring metrics-descriptors list --filter="metric.type:compute"

# Describe a metric
gcloud monitoring metrics-descriptors describe \
    "compute.googleapis.com/instance/cpu/utilization"

# ─── Dashboards ──────────────────────────────────────────────
# Create dashboard
gcloud monitoring dashboards create --config-from-file=dashboard.json

# List dashboards
gcloud monitoring dashboards list

# Describe dashboard
gcloud monitoring dashboards describe DASHBOARD_ID

# Update dashboard
gcloud monitoring dashboards update DASHBOARD_ID --config-from-file=dashboard.json

# Delete dashboard
gcloud monitoring dashboards delete DASHBOARD_ID

# ─── Alerting Policies ───────────────────────────────────────
# Create policy
gcloud alpha monitoring policies create --policy-from-file=policy.json

# List policies
gcloud alpha monitoring policies list

# Describe policy
gcloud alpha monitoring policies describe POLICY_ID

# Update policy
gcloud alpha monitoring policies update POLICY_ID --policy-from-file=policy.json

# Delete policy
gcloud alpha monitoring policies delete POLICY_ID

# Enable/disable
gcloud alpha monitoring policies update POLICY_ID --enabled
gcloud alpha monitoring policies update POLICY_ID --no-enabled

# ─── Notification Channels ───────────────────────────────────
# Create channel
gcloud alpha monitoring channels create \
    --display-name="NAME" --type=TYPE --channel-labels=KEY=VALUE

# List channels
gcloud alpha monitoring channels list

# Delete channel
gcloud alpha monitoring channels delete CHANNEL_ID

# ─── Uptime Checks ───────────────────────────────────────────
# Create uptime check
gcloud monitoring uptime create "NAME" \
    --resource-type=uptime-url \
    --hostname="HOST" \
    --path="/PATH" \
    --protocol=HTTPS

# List uptime checks
gcloud monitoring uptime list-configs

# Delete uptime check
gcloud monitoring uptime delete CHECK_ID

# ─── Services & SLOs ─────────────────────────────────────────
# Create service
gcloud monitoring services create --display-name="NAME" --service-id=ID

# List services
gcloud monitoring services list

# Create SLO
gcloud monitoring slos create --service=SERVICE_ID \
    --display-name="NAME" --goal=0.999 --rolling-period=30d

# List SLOs
gcloud monitoring slos list --service=SERVICE_ID

# ─── Metrics Scope ───────────────────────────────────────────
# Add monitored project to scope
gcloud alpha monitoring metrics-scopes create \
    projects/MONITORED_PROJECT \
    --monitoring-project=SCOPING_PROJECT
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Golden Signals Monitoring

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: GOLDEN SIGNALS (LATENCY, TRAFFIC, ERRORS, SAT)        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Dashboard Layout — "Four Golden Signals"                              │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  ┌─────────────────────┐  ┌─────────────────────┐        │      │
│  │  │ LATENCY             │  │ TRAFFIC             │        │      │
│  │  │ • p50, p95, p99     │  │ • Requests/sec      │        │      │
│  │  │ • By endpoint       │  │ • By method/path    │        │      │
│  │  │ Alert: p99 > 500ms  │  │ Alert: traffic drop │        │      │
│  │  └─────────────────────┘  └─────────────────────┘        │      │
│  │  ┌─────────────────────┐  ┌─────────────────────┐        │      │
│  │  │ ERRORS              │  │ SATURATION          │        │      │
│  │  │ • Error rate %      │  │ • CPU utilization   │        │      │
│  │  │ • By status code    │  │ • Memory usage      │        │      │
│  │  │ Alert: rate > 1%    │  │ • Disk, connections │        │      │
│  │  └─────────────────────┘  │ Alert: > 80%        │        │      │
│  │                            └─────────────────────┘        │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Alert tiers:                                                          │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ Critical (page):  Error rate > 5% | Latency p99 > 2s      │      │
│  │ Warning (ticket): Error rate > 1% | CPU > 80% sustained   │      │
│  │ Info (log):       Traffic anomaly | New error type          │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```hcl
# Golden signals alerts for Cloud Run service
resource "google_monitoring_alert_policy" "latency_p99" {
  display_name = "[Critical] Latency P99 > 2s"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "P99 latency exceeds 2 seconds"
    condition_threshold {
      filter          = "metric.type=\"run.googleapis.com/request_latencies\" resource.type=\"cloud_run_revision\" resource.labels.service_name=\"${var.service_name}\""
      comparison      = "COMPARISON_GT"
      threshold_value = 2000  # milliseconds
      duration        = "300s"
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_PERCENTILE_99"
      }
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.pagerduty.name,
  ]
}

resource "google_monitoring_alert_policy" "error_rate" {
  display_name = "[Critical] Error Rate > 5%"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "Error rate exceeds 5%"
    condition_monitoring_query_language {
      query    = <<-MQL
        fetch cloud_run_revision
        | metric 'run.googleapis.com/request_count'
        | filter resource.service_name = '${var.service_name}'
        | align rate(5m)
        | { filter metric.response_code_class = '5xx' ; ident }
        | group_by [], [sum(val())]
        | ratio
        | condition val() > 0.05 '5m'
      MQL
      duration = "0s"
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.pagerduty.name,
    google_monitoring_notification_channel.slack.name,
  ]
}

resource "google_monitoring_alert_policy" "traffic_drop" {
  display_name = "[Warning] Traffic Drop > 50%"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "Traffic dropped by more than 50%"
    condition_monitoring_query_language {
      query    = <<-MQL
        fetch cloud_run_revision
        | metric 'run.googleapis.com/request_count'
        | filter resource.service_name = '${var.service_name}'
        | align rate(10m)
        | group_by [], [sum(val())]
        | condition val() < 0.5 * val(10m ago) '10m'
      MQL
      duration = "0s"
    }
  }

  notification_channels = [
    google_monitoring_notification_channel.slack.name,
  ]
}
```

### Pattern 2: Multi-Project Observability Hub

```
┌──────────────────────────────────────────────────────────────────────┐
│   PATTERN 2: CENTRALIZED OBSERVABILITY WITH METRICS SCOPES            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  Scoping Project: "observability-hub"                        │     │
│  │                                                              │     │
│  │  Monitored projects:                                         │     │
│  │  ├── prod-app-1        (production services)                │     │
│  │  ├── prod-app-2        (production services)                │     │
│  │  ├── prod-data         (databases, storage)                 │     │
│  │  ├── staging-all       (staging environment)                │     │
│  │  └── shared-infra      (networking, load balancers)         │     │
│  │                                                              │     │
│  │  Centralized resources:                                      │     │
│  │  • All dashboards (cross-project views)                     │     │
│  │  • All alerting policies (unified)                          │     │
│  │  • All uptime checks                                        │     │
│  │  • All notification channels                                │     │
│  │  • SLOs across services                                     │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  Benefits:                                                             │
│  • Single pane of glass across all environments                       │
│  • Cross-project correlation (e.g., LB + backend metrics)             │
│  • Centralized alert management                                       │
│  • Consistent dashboards across teams                                 │
│  • Unified SLO tracking                                               │
│                                                                        │
│  Implementation:                                                       │
│  1. Create dedicated observability project                            │
│  2. Add all projects to its metrics scope                             │
│  3. Create dashboards referencing cross-project metrics               │
│  4. Define alerts in the scoping project                              │
│  5. Grant teams config.viewer on scoping project                      │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```hcl
# Create the observability hub project
module "observability_project" {
  source  = "terraform-google-modules/project-factory/google"
  version = "~> 15.0"

  name            = "observability-hub"
  org_id          = var.org_id
  billing_account = var.billing_account
  
  activate_apis = ["monitoring.googleapis.com"]
}

# Add monitored projects to the metrics scope
resource "google_monitoring_monitored_project" "prod_app_1" {
  metrics_scope = "projects/${module.observability_project.project_id}"
  name          = "projects/prod-app-1"
}

resource "google_monitoring_monitored_project" "prod_app_2" {
  metrics_scope = "projects/${module.observability_project.project_id}"
  name          = "projects/prod-app-2"
}

resource "google_monitoring_monitored_project" "prod_data" {
  metrics_scope = "projects/${module.observability_project.project_id}"
  name          = "projects/prod-data"
}

# Cross-project dashboard
resource "google_monitoring_dashboard" "cross_project" {
  project        = module.observability_project.project_id
  dashboard_json = jsonencode({
    displayName = "Cross-Project Production Overview"
    mosaicLayout = {
      columns = 12
      tiles = [
        {
          xPos = 0, yPos = 0, width = 12, height = 4
          widget = {
            title = "All Projects — Request Rate"
            xyChart = {
              dataSets = [{
                timeSeriesQuery = {
                  timeSeriesFilter = {
                    filter = "metric.type=\"run.googleapis.com/request_count\" resource.type=\"cloud_run_revision\""
                    aggregation = {
                      alignmentPeriod    = "60s"
                      perSeriesAligner   = "ALIGN_RATE"
                      crossSeriesReducer = "REDUCE_SUM"
                      groupByFields      = ["resource.labels.project_id"]
                    }
                  }
                }
                plotType = "STACKED_AREA"
              }]
            }
          }
        }
      ]
    }
  })
}
```

### Pattern 3: SLO-Driven Reliability with Error Budgets

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: SLO-DRIVEN OPERATIONS                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Step 1: Define SLOs per service                                       │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ Service         │ SLI               │ SLO    │ Window     │      │
│  │ checkout-api    │ Availability      │ 99.9%  │ 30 days    │      │
│  │ checkout-api    │ Latency p99       │ 99%    │ 30 days    │      │
│  │ search-service  │ Availability      │ 99.5%  │ 30 days    │      │
│  │ payment-gateway │ Availability      │ 99.99% │ 30 days    │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Step 2: Set up burn rate alerts                                       │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ Alert               │ Burn Rate │ Window │ Action          │      │
│  │ Critical/Page       │ 14.4x     │ 1h     │ PagerDuty      │      │
│  │ Warning/Ticket      │ 6x        │ 6h     │ Jira + Slack   │      │
│  │ Info/Observation    │ 3x        │ 3d     │ Slack           │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Step 3: Error budget dashboard                                        │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ Service         │ Budget │ Consumed │ Remaining │ Status   │      │
│  │ checkout-api    │ 43.2m  │ 12.5m    │ 30.7m     │ ✓ OK    │      │
│  │ search-service  │ 216m   │ 180m     │ 36m       │ ⚠ Warn  │      │
│  │ payment-gateway │ 4.3m   │ 4.1m     │ 0.2m      │ ✗ Crit  │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Step 4: Error budget policy                                           │
│  • Budget > 50%: Deploy freely, experiment                            │
│  • Budget 20-50%: Deploy with caution, extra testing                  │
│  • Budget < 20%: Feature freeze, focus on reliability                 │
│  • Budget exhausted: Incident response, no deploys                    │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command / Location |
|--------|-------------------|
| View metrics | Console → Monitoring → Metrics Explorer |
| Create dashboard | `gcloud monitoring dashboards create --config-from-file=FILE` |
| Create alert | `gcloud alpha monitoring policies create --policy-from-file=FILE` |
| Create uptime check | `gcloud monitoring uptime create "NAME" --hostname=HOST --path=PATH` |
| List alerts | `gcloud alpha monitoring policies list` |
| Create notification channel | `gcloud alpha monitoring channels create --type=TYPE` |
| Add to metrics scope | `gcloud alpha monitoring metrics-scopes create projects/PROJ` |
| Write custom metric | Monitoring API `CreateTimeSeries` |
| Query with MQL | Console → Metrics Explorer → MQL tab |
| Query with PromQL | Console → Metrics Explorer → PromQL tab |
| Create SLO | `gcloud monitoring slos create --service=SVC --goal=0.999` |
| View dashboards | Console → Monitoring → Dashboards |

---

## What is Monitoring? (Beginner Explanation)

### The Hospital Analogy

Think of **monitoring as a health checkup for your servers**. Just like a doctor monitors a patient's vital signs, Cloud Monitoring continuously checks the "health" of your applications and infrastructure.

```
┌──────────────────────────────────────────────────────────────┐
│         MONITORING = HOSPITAL FOR YOUR SERVERS                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Patient (Server)          Hospital (Monitoring)              │
│  ─────────────────         ─────────────────────              │
│                                                                │
│  Heart Rate        ═══►   CPU Utilization                     │
│  (beats/min)               (% of processing power used)       │
│                                                                │
│  Blood Pressure    ═══►   Memory Usage                        │
│  (pressure level)          (% of RAM consumed)                │
│                                                                │
│  Temperature       ═══►   Error Rate                          │
│  (body temp)               (% of requests failing)            │
│                                                                │
│  Breathing Rate    ═══►   Network Traffic                     │
│  (breaths/min)             (bytes in/out per second)          │
│                                                                │
│  Blood Oxygen      ═══►   Disk Utilization                    │
│  (O₂ saturation)           (% of storage used)                │
│                                                                │
│  Patient Monitor   ═══►   Dashboard                           │
│  (the screen)              (visual display of all metrics)    │
│                                                                │
│  Alarm Beep        ═══►   Alert                               │
│  (wakes the doctor)        (sends notification to engineer)   │
│                                                                │
│  Nurse Check       ═══►   Uptime Check                        │
│  ("Are you okay?")         ("Is the service responding?")     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Breaking It Down

| Concept | Hospital Analogy | What It Really Means |
|---------|-----------------|---------------------|
| **Metrics** | Vital signs (heart rate, BP, temp) | Numbers that describe your system's behaviour — CPU usage, request count, latency |
| **Alerts** | The alarm that wakes the doctor when something is wrong | Automatic notifications (email, Slack, PagerDuty) when a metric crosses a threshold |
| **Dashboards** | The patient monitor screen showing all vitals at once | A visual display that shows multiple metrics in charts and graphs |
| **Uptime Checks** | The nurse who walks in every 15 min to ask "Are you okay?" | Probes from global locations that hit your service to confirm it's alive |
| **Notification Channels** | The pager the doctor carries | Where alerts are sent — email, Slack, SMS, PagerDuty, webhooks |
| **SLOs** | "Patient must maintain heart rate between 60–100 bpm" | "Service must be available 99.9% of the time" — your reliability targets |

### Why Does Monitoring Matter?

1. **Detect problems before users do** — An alert fires at 3 AM when CPU spikes, so you fix it before customers notice
2. **Understand performance trends** — "Memory usage has been creeping up 2% per week" tells you it's time to scale
3. **Speed up debugging** — When something breaks, dashboards show exactly *when* and *where* things went wrong
4. **Prove reliability** — SLOs give you a data-driven answer to "How reliable is our service?"
5. **Plan capacity** — Metrics show you when you're running out of resources, so you can scale before it's too late

> **One-liner**: If your servers are patients, Cloud Monitoring is the entire hospital — the monitors, the alarms, the nurses, and the pagers — all working 24/7 so you can sleep at night.

---

## Console Walkthrough: Deleting Resources

### Deleting Alert Policies

```
┌──────────────────────────────────────────────────────────────┐
│         DELETE ALERT POLICIES FROM CONSOLE                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Monitoring → Alerting                              │
│                                                                │
│  Step 2: Find the policy                                       │
│  • You'll see a list of all alerting policies                 │
│  • Use the search/filter bar to find a specific policy        │
│                                                                │
│  Step 3: Delete                                                │
│  • Click on the policy name to open it                        │
│  • Click the ⋮ (three-dot menu) in the top-right corner      │
│  • Select "Delete"                                            │
│  • Confirm the deletion in the dialog                         │
│                                                                │
│  Alternative (bulk):                                           │
│  • Check the checkbox next to one or more policies            │
│  • Click "Delete" in the top action bar                       │
│  • Confirm the deletion                                       │
│                                                                │
│  ⚠ Deleting a policy also resolves any open incidents         │
│    associated with it.                                         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud alpha monitoring policies list
gcloud alpha monitoring policies delete POLICY_ID
```

### Deleting Dashboards

```
┌──────────────────────────────────────────────────────────────┐
│         DELETE DASHBOARDS FROM CONSOLE                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Monitoring → Dashboards                            │
│                                                                │
│  Step 2: Find the dashboard                                    │
│  • "Custom" tab shows your user-created dashboards            │
│  • "Predefined" tab shows auto-generated dashboards (cannot   │
│    be deleted)                                                │
│                                                                │
│  Step 3: Delete                                                │
│  • Click the ⋮ (three-dot menu) next to the dashboard name   │
│  • Select "Delete Dashboard"                                  │
│  • Confirm the deletion                                       │
│                                                                │
│  Note: Predefined (GCP auto-generated) dashboards cannot be   │
│  deleted — they are auto-managed by GCP.                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud monitoring dashboards list
gcloud monitoring dashboards delete DASHBOARD_ID
```

### Deleting Notification Channels

```
┌──────────────────────────────────────────────────────────────┐
│         DELETE NOTIFICATION CHANNELS FROM CONSOLE             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Monitoring → Alerting → "Edit notification         │
│  channels" (link at the top of the Alerting page)             │
│                                                                │
│  Step 2: Find the channel                                      │
│  • Channels are grouped by type (Email, Slack, PagerDuty,     │
│    Webhooks, SMS, Pub/Sub, etc.)                              │
│  • Expand the section for the channel type                    │
│                                                                │
│  Step 3: Delete                                                │
│  • Click the 🗑 (delete/trash icon) next to the channel       │
│  • Confirm the deletion                                       │
│                                                                │
│  ⚠ Before deleting, check if the channel is used by any      │
│    alerting policies. Deleting a channel that is referenced    │
│    by a policy will cause that policy to have no notification  │
│    target — alerts will fire but nobody will be notified!      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud alpha monitoring channels list
gcloud alpha monitoring channels delete CHANNEL_ID

# Force delete (even if referenced by policies)
gcloud alpha monitoring channels delete CHANNEL_ID --force
```

### Deleting Uptime Checks

```
┌──────────────────────────────────────────────────────────────┐
│         DELETE UPTIME CHECKS FROM CONSOLE                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Monitoring → Uptime checks                         │
│                                                                │
│  Step 2: Find the uptime check                                │
│  • You'll see a list of all configured uptime checks          │
│  • Each shows status (passing/failing), target, and interval  │
│                                                                │
│  Step 3: Delete                                                │
│  • Click on the uptime check name to open its details         │
│  • Click the ⋮ (three-dot menu) in the top-right corner      │
│  • Select "Delete"                                            │
│  • Confirm the deletion                                       │
│                                                                │
│  ⚠ If the uptime check has an associated alerting policy,     │
│    you'll be asked whether to also delete that policy.         │
│    If you don't delete the policy, it will still exist but     │
│    won't have a valid condition (and won't fire).              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud monitoring uptime list-configs
gcloud monitoring uptime delete CHECK_ID
```

---

## What's Next?

Continue to **Chapter 37: Cloud Logging** → `37-cloud-logging.md`
