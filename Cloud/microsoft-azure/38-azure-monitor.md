# Chapter 38: Azure Monitor

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Monitor Fundamentals](#part-1-azure-monitor-fundamentals)
- [Part 2: Metrics](#part-2-metrics)
- [Part 3: Alerts & Action Groups](#part-3-alerts--action-groups)
- [Part 4: Dashboards](#part-4-dashboards)
- [Part 5: Diagnostic Settings](#part-5-diagnostic-settings)
- [Part 6: Autoscale with Metrics](#part-6-autoscale-with-metrics)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Monitor is the central monitoring service for everything in Azure. It collects metrics (numbers like CPU%, memory) and logs (text events) from all your Azure resources, lets you set alerts, create dashboards, and automatically respond to issues.

```
What you'll learn:
├── Azure Monitor Fundamentals
│   ├── What is Azure Monitor (unified monitoring platform)
│   ├── Metrics vs Logs
│   └── Data sources (Azure resources, VMs, apps, custom)
├── Metrics (explore, chart, pin to dashboard)
├── Alerts & Action Groups (email, SMS, webhook)
├── Dashboards (visualize everything)
├── Diagnostic Settings (send data to destinations)
├── Autoscale with Metrics
├── Terraform, Bicep, az CLI
└── Quick reference
```

---

## Part 1: Azure Monitor Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE MONITOR OVERVIEW                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure Monitor = Central monitoring hub for ALL Azure resources    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────┐          │
│ │ Data Sources          │  Azure Monitor               │          │
│ │                       │                               │          │
│ │ Azure Resources ──────┤  ┌─────────┐ ┌──────────┐  │          │
│ │ (VMs, SQL, App Svc)  │  │ Metrics │ │ Logs     │  │          │
│ │                       │  │ (numbers)│ │ (text)   │  │          │
│ │ Applications ─────────┤  └────┬────┘ └────┬─────┘  │          │
│ │ (App Insights)       │       │            │         │          │
│ │                       │  ┌────▼────────────▼─────┐  │          │
│ │ OS (VM Agent) ────────┤  │ Alerts, Dashboards,  │  │          │
│ │                       │  │ Workbooks, Autoscale  │  │          │
│ │ Custom (API) ─────────┤  └──────────────────────┘  │          │
│ └──────────────────────────────────────────────────────┘          │
│                                                                       │
│ Metrics vs Logs:                                                     │
│ ┌──────────────┬────────────────────┬────────────────────┐        │
│ │              │ Metrics            │ Logs               │        │
│ ├──────────────┼────────────────────┼────────────────────┤        │
│ │ What         │ Numbers over time  │ Text events        │        │
│ │ Example      │ CPU: 75%          │ "User logged in"   │        │
│ │ Storage      │ Time-series DB    │ Log Analytics       │        │
│ │ Retention    │ 93 days (free)    │ 30-730 days (paid)│        │
│ │ Query        │ Metrics Explorer  │ KQL (Kusto)        │        │
│ │ Cost         │ Free (platform)   │ Pay per GB ingested│        │
│ │ Latency      │ Near real-time    │ Minutes            │        │
│ └──────────────┴────────────────────┴────────────────────┘        │
│                                                                       │
│ ⚡ Platform metrics (CPU, memory, disk) are collected for free! │
│ ⚡ Logs (Log Analytics) cost money per GB ingested.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Metrics

```
┌─────────────────────────────────────────────────────────────────────┐
│           METRICS EXPLORER                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Monitor → Metrics                                       │
│ OR: Any resource → Monitoring → Metrics                           │
│                                                                       │
│ Scope: [myapp-vm ▼] (select resource)                             │
│ Metric Namespace: [Virtual Machine Host ▼]                        │
│ Metric: [Percentage CPU ▼]                                        │
│ Aggregation: [Avg ▼] (Avg/Min/Max/Sum/Count)                    │
│ Time range: [Last 24 hours ▼]                                    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────┐          │
│ │ CPU %                                                  │          │
│ │ 100│                                                   │          │
│ │  80│        ╱╲                                        │          │
│ │  60│       ╱  ╲    ╱╲                                │          │
│ │  40│      ╱    ╲  ╱  ╲                              │          │
│ │  20│     ╱      ╲╱    ╲                             │          │
│ │   0│────╱─────────────────                           │          │
│ │    └──────────────────────────────                   │          │
│ │     6AM  9AM  12PM  3PM  6PM  9PM                   │          │
│ └──────────────────────────────────────────────────────┘          │
│                                                                       │
│ Actions:                                                             │
│ [+ Add metric] → Overlay multiple metrics on same chart           │
│ [Pin to dashboard] → Add chart to Azure Dashboard                │
│ [New alert rule] → Create alert from this metric                 │
│ [Share] → Get link to this chart view                            │
│                                                                       │
│ Common metrics by resource:                                         │
│ ├── VM: CPU %, Memory %, Disk Read/Write, Network In/Out        │
│ ├── App Service: Requests, Response Time, HTTP errors, CPU      │
│ ├── SQL Database: DTU %, CPU %, Storage, Deadlocks              │
│ ├── Storage: Transactions, Ingress, Egress, Latency             │
│ └── Load Balancer: Health probe status, data path availability  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Alerts & Action Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│           CREATE ALERT RULE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Monitor → Alerts → Create → Alert rule                           │
│                                                                       │
│ ── Scope ──                                                         │
│ Resource: [myapp-vm ▼]                                             │
│                                                                       │
│ ── Condition ──                                                     │
│ Signal type: [Metrics ▼]                                          │
│ Signal name: [Percentage CPU ▼]                                   │
│                                                                       │
│ Alert logic:                                                         │
│ Threshold: [Static ▼]                                             │
│ Operator: [Greater than ▼]                                        │
│ Threshold value: [80] %                                            │
│ Aggregation type: [Average ▼]                                     │
│ Aggregation granularity: [5 minutes ▼]                            │
│ Frequency of evaluation: [Every 1 minute ▼]                      │
│                                                                       │
│ ⚡ This means: "Alert if average CPU > 80% for 5 minutes"       │
│                                                                       │
│ ── Action Group ──                                                  │
│ [Create action group]                                               │
│ Name: [ag-ops-team]                                                │
│                                                                       │
│ Notifications:                                                       │
│ ├── Email: [ops@company.com] ☑                                  │
│ ├── SMS: [+91-XXXXXXXXXX] ☑                                    │
│ ├── Push notification: ☑ (Azure mobile app)                     │
│ └── Voice call: ☐                                                │
│                                                                       │
│ Actions:                                                             │
│ ├── Webhook: [https://hooks.slack.com/...] (Slack notification)│
│ ├── Azure Function: [func-auto-scale]                            │
│ ├── Logic App: [la-incident-workflow]                            │
│ ├── Automation Runbook: [Scale up VM]                            │
│ └── ITSM: [ServiceNow connector]                                │
│                                                                       │
│ ── Details ──                                                       │
│ Severity: [2 - Warning ▼]                                        │
│ Alert rule name: [High CPU Alert]                                  │
│ Description: [CPU above 80% for 5 minutes]                       │
│                                                                       │
│ Severity levels:                                                     │
│ 0 = Critical  (system down)                                       │
│ 1 = Error     (service degraded)                                  │
│ 2 = Warning   (approaching limit)                                 │
│ 3 = Informational                                                  │
│ 4 = Verbose                                                        │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
Alert types:
├── Metric alerts → Numbers exceed threshold
│   Example: CPU > 80%, memory > 90%, response time > 2s
├── Log alerts → KQL query returns results
│   Example: Count of 500 errors in last 5 minutes > 10
├── Activity log alerts → Azure management events
│   Example: VM deleted, resource group modified
└── Smart detection → AI detects anomalies (App Insights)
```

---

## Part 4: Dashboards

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE DASHBOARDS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Dashboard → [+ New dashboard]                           │
│                                                                       │
│ Name: [Operations Dashboard]                                       │
│                                                                       │
│ Available tiles (drag & drop):                                      │
│ ├── Metrics chart (pin from Metrics Explorer)                    │
│ ├── Resource health                                                │
│ ├── Alert summary                                                  │
│ ├── Markdown (custom text/links)                                 │
│ ├── Clock                                                          │
│ ├── Azure Resource Graph query                                   │
│ └── Application map (App Insights)                                │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐  │
│ │ 🔧 Operations Dashboard                                      │  │
│ │                                                               │  │
│ │ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │  │
│ │ │ VM CPU %     │ │ App Requests │ │ DB DTU %     │       │  │
│ │ │ ▓▓▓▓▓▓ 45%  │ │ ████ 1.2k/m │ │ ▓▓▓ 30%     │       │  │
│ │ └──────────────┘ └──────────────┘ └──────────────┘       │  │
│ │ ┌──────────────┐ ┌──────────────────────────────┐         │  │
│ │ │ Active Alerts │ │ Response Time (p95)          │         │  │
│ │ │ 🔴 2 Critical │ │ ──────╱╲─────── 180ms      │         │  │
│ │ │ 🟡 5 Warning  │ │                              │         │  │
│ │ └──────────────┘ └──────────────────────────────┘         │  │
│ └─────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Sharing:                                                             │
│ ├── Private (only you)                                            │
│ ├── Shared (anyone in subscription with Dashboard Reader role)  │
│ └── Export as JSON → Import on another subscription             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Diagnostic Settings

```
┌─────────────────────────────────────────────────────────────────────┐
│           DIAGNOSTIC SETTINGS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Any resource → Monitoring → Diagnostic settings → [+ Add]       │
│                                                                       │
│ Name: [send-to-log-analytics]                                      │
│                                                                       │
│ Logs:                                                                │
│ ☑ AllLogs  OR  select specific categories                        │
│                                                                       │
│ Metrics:                                                             │
│ ☑ AllMetrics                                                      │
│                                                                       │
│ Destination:                                                         │
│ ☑ Send to Log Analytics workspace: [law-myapp-prod ▼]          │
│ ☐ Archive to a storage account                                   │
│ ☐ Stream to an event hub                                         │
│ ☐ Send to partner solution                                       │
│                                                                       │
│ ⚡ Diagnostic settings tell Azure WHERE to send monitoring data. │
│ ⚡ Without this, detailed logs are NOT collected!                │
│ ⚡ Platform metrics are free; diagnostic logs cost per GB.       │
│                                                                       │
│ Common setup:                                                        │
│ ├── Send all resource logs → Log Analytics (for querying)       │
│ ├── Archive old logs → Storage Account (cheaper long-term)      │
│ └── Stream real-time → Event Hub (for SIEM, Splunk, Datadog)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Autoscale with Metrics

```
Resource → Scaling → Custom autoscale

Scale based on metric:
├── Metric: CPU Percentage
├── Scale out: When CPU > 70% for 10 min → Add 1 instance
├── Scale in: When CPU < 30% for 10 min → Remove 1 instance
├── Min instances: 2
├── Max instances: 10
└── Default: 2

⚡ Azure Monitor metrics drive autoscaling decisions!
⚡ Covered in detail in App Service (ch 20) and VM Scale Sets (ch 21)
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
# Metric alert
resource "azurerm_monitor_metric_alert" "cpu" {
  name                = "high-cpu-alert"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_linux_virtual_machine.main.id]
  severity            = 2
  frequency           = "PT1M"
  window_size         = "PT5M"

  criteria {
    metric_namespace = "Microsoft.Compute/virtualMachines"
    metric_name      = "Percentage CPU"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = azurerm_monitor_action_group.ops.id
  }
}

resource "azurerm_monitor_action_group" "ops" {
  name                = "ag-ops-team"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "ops"

  email_receiver {
    name          = "ops-email"
    email_address = "ops@company.com"
  }
}
```

### Bicep

```bicep
resource actionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'ag-ops-team'
  location: 'global'
  properties: {
    groupShortName: 'ops'
    enabled: true
    emailReceivers: [
      {
        name: 'ops-email'
        emailAddress: 'ops@company.com'
      }
    ]
  }
}
```

---

## Part 8: az CLI Reference

```bash
# List metrics for a resource
az monitor metrics list \
  --resource /subscriptions/.../virtualMachines/myvm \
  --metric "Percentage CPU" \
  --interval PT1H

# Create action group
az monitor action-group create \
  --name ag-ops-team \
  --resource-group rg-monitoring \
  --short-name ops \
  --email-receiver ops ops@company.com

# Create metric alert
az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group rg-monitoring \
  --scopes /subscriptions/.../virtualMachines/myvm \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action ag-ops-team

# List alerts
az monitor metrics alert list --resource-group rg-monitoring --output table

# Create diagnostic setting
az monitor diagnostic-settings create \
  --name send-to-law \
  --resource /subscriptions/.../sites/myapp \
  --workspace /subscriptions/.../workspaces/law-myapp \
  --logs '[{"category":"AppServiceHTTPLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Delete alert
az monitor metrics alert delete --name "High CPU Alert" --resource-group rg-monitoring
```

---

## Real-World Patterns

### Pattern 1: Full-Stack Observability

```
┌─────────────────────────────────────────────────┐
│       Observability Architecture                │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ App Svc │  │ AKS     │  │ SQL DB  │        │
│  └────┬────┘  └────┬────┘  └────┬────┘        │
│       │          │          │               │
│       └──────────┼──────────┘               │
│                  ▼                              │
│        ┌──────────────────┐                     │
│        │  Azure Monitor    │                     │
│        ├──────────────────┤                     │
│        │ Metrics + Logs   │                     │
│        │ Alerts           │                     │
│        │ Dashboards       │                     │
│        │ Workbooks        │                     │
│        └───────┬──────────┘                     │
│                ▼                                │
│        Action Groups: Email, SMS, Logic App     │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure Monitor = Central monitoring for all Azure resources

Data types:
  Metrics = Numbers (CPU%, memory, requests) — free, real-time
  Logs = Text events — paid per GB, query with KQL

Alerts → Action Groups → Notifications (email/SMS/webhook)
Severity: 0=Critical, 1=Error, 2=Warning, 3=Info, 4=Verbose

Diagnostic Settings = Tell Azure where to send detailed logs
Destinations: Log Analytics, Storage Account, Event Hub

Dashboards: Pin metrics charts, share with team
Autoscale: Scale resources based on metric thresholds
```

---

## What's Next?

Next chapter: [Chapter 39: Log Analytics & KQL](39-log-analytics.md) — Log Analytics workspaces, Kusto Query Language, and advanced log analysis.
