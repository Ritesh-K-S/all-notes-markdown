# Chapter 39: Log Analytics & KQL

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Log Analytics Fundamentals](#part-1-log-analytics-fundamentals)
- [Part 2: Creating a Log Analytics Workspace (Portal Walkthrough)](#part-2-creating-a-log-analytics-workspace-portal-walkthrough)
- [Part 3: KQL (Kusto Query Language)](#part-3-kql-kusto-query-language)
- [Part 4: Common Log Tables](#part-4-common-log-tables)
- [Part 5: Log Alerts](#part-5-log-alerts)
- [Part 6: Workbooks](#part-6-workbooks)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Log Analytics is where all your logs go in Azure. It uses workspaces to store log data and KQL (Kusto Query Language) to query it. Think of it as a powerful database for searching through millions of log entries to find problems, patterns, and insights.

```
What you'll learn:
├── Log Analytics Fundamentals
│   ├── What is a workspace (centralized log store)
│   ├── Data ingestion (what sends logs here)
│   └── Retention & pricing
├── Creating a Workspace (Portal)
├── KQL (Kusto Query Language) — essential queries
├── Common Log Tables (where data lives)
├── Log Alerts (alert on query results)
├── Workbooks (rich visualizations)
├── Terraform, Bicep, az CLI
└── Quick reference
```

---

## Part 1: Log Analytics Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           LOG ANALYTICS OVERVIEW                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Log Analytics?                                               │
│ ├── A centralized workspace for storing log data                 │
│ ├── All Azure services can send logs here                        │
│ ├── Query logs using KQL (Kusto Query Language)                  │
│ └── Powers: Alerts, Workbooks, Azure Sentinel, Defender         │
│                                                                       │
│ What sends data to Log Analytics?                                   │
│ ├── Azure resources (via Diagnostic Settings)                    │
│ ├── VMs (via Azure Monitor Agent - AMA)                          │
│ ├── Application Insights (app telemetry)                         │
│ ├── Microsoft Defender for Cloud (security events)              │
│ ├── Azure Activity Log (management operations)                  │
│ └── Custom logs (API, Fluentd, Logstash)                        │
│                                                                       │
│ Pricing:                                                             │
│ ├── Pay-as-you-go: ~$2.76/GB ingested (varies by region)       │
│ ├── Commitment tiers: 100GB/day+ for discounts                  │
│ ├── Retention: 30 days free, 31-730 days paid                   │
│ └── First 5 GB/month free (per billing account)                 │
│                                                                       │
│ Architecture:                                                        │
│ ┌─────────────────────────────────────────────────────┐          │
│ │ Log Analytics Workspace: law-myapp-prod              │          │
│ │                                                       │          │
│ │ ┌──────────┐ ┌──────────┐ ┌──────────────────┐    │          │
│ │ │ Tables   │ │ Tables   │ │ Tables            │    │          │
│ │ │ AzureAct │ │ Heartbeat│ │ AppServiceHTTPLogs│    │          │
│ │ │ Syslog   │ │ Perf     │ │ ContainerLog      │    │          │
│ │ │ Event    │ │ AzureMetr│ │ AppRequests       │    │          │
│ │ └──────────┘ └──────────┘ └──────────────────┘    │          │
│ │                                                       │          │
│ │ Query with KQL → Results → Alerts/Workbooks       │          │
│ └─────────────────────────────────────────────────────┘          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Log Analytics Workspace (Portal Walkthrough)

```
Console → Log Analytics workspaces → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE WORKSPACE                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-monitoring ▼]                                  │
│                                                                       │
│ Name: [law-myapp-prod]                                              │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ ── Pricing Tier ──                                                  │
│ Pricing tier: [Pay-as-you-go ▼]                                   │
│ ├── Pay-as-you-go (default, good for < 100 GB/day)              │
│ ├── Commitment Tier 100 GB/day (discount)                        │
│ ├── Commitment Tier 200 GB/day                                   │
│ └── Commitment Tier 300+ GB/day                                  │
│                                                                       │
│ Data retention: [30] days (free)                                   │
│ ⚡ 30 days free. 31-730 days: ~$0.10/GB/month                  │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation, connect data sources:                               │
│ 1. Azure resources → Diagnostic Settings → Send to workspace   │
│ 2. VMs → Install Azure Monitor Agent                             │
│ 3. Activity Log → Monitor → Activity log → Export              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: KQL (Kusto Query Language)

```
┌─────────────────────────────────────────────────────────────────────┐
│           KQL BASICS                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Go to: Log Analytics workspace → Logs                             │
│                                                                       │
│ KQL reads like English, left to right, top to bottom.             │
│ Each line is a "pipe" that transforms the data.                   │
│                                                                       │
│ Basic structure:                                                     │
│ TableName                                                            │
│ | where <condition>                                                │
│ | project <columns>                                                │
│ | sort by <column>                                                 │
│ | take <number>                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Essential KQL Queries

```kql
// 1. See all data in a table (last 24 hours by default)
AzureActivity
| take 10

// 2. Filter by time
AzureActivity
| where TimeGenerated > ago(1h)

// 3. Filter by condition
AzureActivity
| where OperationNameValue == "MICROSOFT.COMPUTE/VIRTUALMACHINES/WRITE"
| where ActivityStatusValue == "Success"

// 4. Select specific columns
AzureActivity
| where TimeGenerated > ago(24h)
| project TimeGenerated, OperationNameValue, Caller, ResourceGroup

// 5. Count events
AzureActivity
| where TimeGenerated > ago(7d)
| summarize count() by OperationNameValue
| sort by count_ desc
| take 10

// 6. Count over time (chart-friendly)
AppRequests
| where TimeGenerated > ago(24h)
| summarize RequestCount = count() by bin(TimeGenerated, 1h)
| render timechart

// 7. Average response time
AppRequests
| where TimeGenerated > ago(1h)
| summarize AvgDuration = avg(DurationMs) by bin(TimeGenerated, 5m)
| render timechart

// 8. Find errors (HTTP 5xx)
AppRequests
| where TimeGenerated > ago(1h)
| where ResultCode startswith "5"
| project TimeGenerated, Name, ResultCode, DurationMs
| sort by TimeGenerated desc

// 9. Search across all columns
search "error"
| where TimeGenerated > ago(1h)
| take 50

// 10. Join tables
AppRequests
| where TimeGenerated > ago(1h)
| join kind=inner (
    AppExceptions
    | where TimeGenerated > ago(1h)
) on OperationId
| project TimeGenerated, RequestName = Name, ExceptionType = Type, ExceptionMessage = Message

// 11. Percentiles
AppRequests
| where TimeGenerated > ago(1h)
| summarize
    p50 = percentile(DurationMs, 50),
    p95 = percentile(DurationMs, 95),
    p99 = percentile(DurationMs, 99)

// 12. Dynamic alerts — anomaly detection
AppRequests
| where TimeGenerated > ago(7d)
| make-series RequestCount = count() on TimeGenerated step 1h
| extend anomalies = series_decompose_anomalies(RequestCount)
```

```
KQL cheat sheet:
├── where → Filter rows (like SQL WHERE)
├── project → Select columns (like SQL SELECT)
├── summarize → Aggregate (like SQL GROUP BY)
├── sort by → Order results
├── take → Limit results (like SQL TOP/LIMIT)
├── extend → Add computed column
├── join → Combine tables
├── render → Visualize (timechart, barchart, piechart)
├── bin() → Group time into buckets
├── ago() → Relative time (ago(1h), ago(7d))
├── count() → Count rows
├── avg(), sum(), min(), max() → Aggregations
├── percentile() → P50, P95, P99
└── search → Search text across all tables
```

---

## Part 4: Common Log Tables

```
Table                     │ What it contains
──────────────────────────┼─────────────────────────────────────
AzureActivity             │ Azure management operations (who did what)
AzureMetrics              │ Resource metrics (CPU, memory, etc.)
Heartbeat                 │ VM/agent health (is it alive?)
Perf                      │ VM performance counters
Event                     │ Windows Event Log entries
Syslog                    │ Linux syslog entries
SecurityEvent             │ Windows security audit logs
AppRequests               │ Application Insights HTTP requests
AppExceptions             │ Application Insights exceptions
AppTraces                 │ Application Insights trace logs
AppDependencies           │ Application Insights external calls
ContainerLog              │ Container stdout/stderr output
KubeEvents                │ Kubernetes events
AzureDiagnostics          │ Diagnostic logs from Azure resources
SigninLogs                │ Azure AD/Entra sign-in events
AuditLogs                 │ Azure AD/Entra audit events
```

---

## Part 5: Log Alerts

```
┌─────────────────────────────────────────────────────────────────────┐
│           LOG-BASED ALERTS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Monitor → Alerts → Create → Alert rule                           │
│ Signal type: Custom log search                                     │
│                                                                       │
│ Query:                                                               │
│ AppRequests                                                         │
│ | where TimeGenerated > ago(5m)                                   │
│ | where ResultCode startswith "5"                                 │
│ | summarize ErrorCount = count()                                  │
│                                                                       │
│ Alert logic:                                                         │
│ Measure: Table rows                                                │
│ Threshold: Greater than [10]                                       │
│ Frequency: Every [5] minutes                                      │
│ Lookback: [5] minutes                                              │
│                                                                       │
│ ⚡ Fires alert if more than 10 HTTP 500 errors in 5 minutes.    │
│                                                                       │
│ Action group: [ag-ops-team] (email + Slack webhook)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Workbooks

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE WORKBOOKS                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Monitor → Workbooks → [+ New]                                    │
│                                                                       │
│ Workbooks = Rich, interactive reports using KQL queries            │
│                                                                       │
│ Components you can add:                                              │
│ ├── Text (Markdown) → Explanations, headers                     │
│ ├── Query → KQL query → Table, Chart, Grid                    │
│ ├── Parameters → Dropdowns for filtering (time, resource)       │
│ ├── Links → Navigation between sections                         │
│ └── Groups → Organize sections                                  │
│                                                                       │
│ vs Dashboards:                                                       │
│ ├── Dashboards: Quick overview, pin metrics charts               │
│ └── Workbooks: Detailed reports, interactive, KQL-powered       │
│                                                                       │
│ Built-in templates:                                                  │
│ ├── VM Performance                                                │
│ ├── Failure Analysis                                              │
│ ├── Storage Account Overview                                     │
│ └── AKS Cluster Monitoring                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# Send App Service logs to workspace
resource "azurerm_monitor_diagnostic_setting" "app" {
  name                       = "send-to-law"
  target_resource_id         = azurerm_linux_web_app.main.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  enabled_log {
    category = "AppServiceHTTPLogs"
  }

  metric {
    category = "AllMetrics"
  }
}
```

### Bicep

```bicep
resource law 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'law-myapp-prod'
  location: resourceGroup().location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}
```

---

## Part 8: az CLI Reference

```bash
# Create workspace
az monitor log-analytics workspace create \
  --resource-group rg-monitoring \
  --workspace-name law-myapp-prod \
  --location centralindia \
  --retention-time 30

# Run KQL query
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "AzureActivity | take 10"

# List workspaces
az monitor log-analytics workspace list --resource-group rg-monitoring --output table

# Show workspace details
az monitor log-analytics workspace show \
  --resource-group rg-monitoring \
  --workspace-name law-myapp-prod

# List tables in workspace
az monitor log-analytics workspace table list \
  --resource-group rg-monitoring \
  --workspace-name law-myapp-prod --output table

# Delete workspace
az monitor log-analytics workspace delete \
  --resource-group rg-monitoring \
  --workspace-name law-myapp-prod --yes
```

---

## Real-World Patterns

### Pattern 1: Centralized Log Management

```
┌─────────────────────────────────────────────────┐
│  Multi-Source Log Aggregation                   │
├─────────────────────────────────────────────────┤
│                                                 │
│  VMs (Agent) ─────┐                             │
│  AKS (Diag)  ─────┤                             │
│  App Svc (Diag)───┤──→ Log Analytics Workspace  │
│  NSG Flow Logs────┤     │                      │
│  Activity Log ────┘     │                      │
│                         ▼                      │
│              ┌─────────────────┐                │
│              │ KQL Queries     │                │
│              │ Saved Searches  │                │
│              │ Alert Rules     │                │
│              │ Workbooks       │                │
│              └─────────────────┘                │
│                                                 │
│  Retention: 90 days hot, archive to Storage     │
│  Cost tip: Use Basic logs for high-volume data  │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Log Analytics Workspace = Centralized log storage
KQL (Kusto Query Language) = SQL-like language for querying logs

Pricing: ~$2.76/GB ingested, 30 days retention free
Free tier: 5 GB/month

Key KQL operators: where, project, summarize, sort by, take, render
Time: ago(1h), ago(7d), ago(30d)
Aggregate: count(), avg(), sum(), percentile()
Visualize: render timechart, render barchart

Common tables: AzureActivity, AppRequests, Heartbeat, Perf, Syslog
Connect data: Diagnostic Settings → Send to workspace

Log Alerts: KQL query + threshold → Action Group
Workbooks: Interactive reports powered by KQL
```

---

## What's Next?

Next chapter: [Chapter 40: Application Insights](40-application-insights.md) — Application performance monitoring, distributed tracing, and availability tests.
