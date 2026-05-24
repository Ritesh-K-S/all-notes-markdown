# Chapter 40: Application Insights

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Application Insights Fundamentals](#part-1-application-insights-fundamentals)
- [Part 2: Setting Up Application Insights (Portal Walkthrough)](#part-2-setting-up-application-insights-portal-walkthrough)
- [Part 3: Auto-Instrumentation vs SDK](#part-3-auto-instrumentation-vs-sdk)
- [Part 4: Key Features](#part-4-key-features)
- [Part 5: Distributed Tracing](#part-5-distributed-tracing)
- [Part 6: Availability Tests](#part-6-availability-tests)
- [Part 7: Smart Detection & Alerts](#part-7-smart-detection--alerts)
- [Part 8: Terraform & Bicep](#part-8-terraform--bicep)
- [Part 9: az CLI Reference](#part-9-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Application Insights is Azure's Application Performance Monitoring (APM) tool. It monitors your live applications — tracking requests, failures, response times, dependencies, and user behavior. It automatically detects performance anomalies and helps you diagnose issues with distributed tracing.

```
What you'll learn:
├── Application Insights Fundamentals
│   ├── What is APM (Application Performance Monitoring)
│   ├── What data is collected
│   └── Workspace-based vs Classic
├── Setting Up Application Insights (Portal)
├── Auto-Instrumentation vs SDK (code vs codeless)
├── Key Features (Live Metrics, Application Map, Failures, Performance)
├── Distributed Tracing (track requests across microservices)
├── Availability Tests (is my app up?)
├── Smart Detection & Alerts
├── Terraform, Bicep, az CLI
└── Quick reference
```

---

## Part 1: Application Insights Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           APPLICATION INSIGHTS OVERVIEW                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What does Application Insights monitor?                              │
│                                                                       │
│ ┌──────────────────────────────────────────────────────┐          │
│ │ Your Application                                       │          │
│ │                                                        │          │
│ │ Requests ──────→ How many? How fast? Any errors?     │          │
│ │ Dependencies ──→ SQL queries, HTTP calls, Redis      │          │
│ │ Exceptions ───→ Stack traces, error details          │          │
│ │ Page Views ───→ Browser performance, load times      │          │
│ │ Custom Events ─→ Business metrics (orders, signups)  │          │
│ │ Traces ────────→ Custom log messages                  │          │
│ │ Performance ──→ CPU, memory, response times          │          │
│ └──────────────────────────────────────────────────────┘          │
│                                                                       │
│ All data flows to → Log Analytics workspace → Query with KQL    │
│                                                                       │
│ Supported languages:                                                 │
│ ├── .NET / .NET Core (best support)                              │
│ ├── Java                                                          │
│ ├── Node.js / JavaScript                                         │
│ ├── Python                                                        │
│ └── Any language via REST API                                    │
│                                                                       │
│ Workspace-based (recommended, new default):                        │
│ ├── Data stored in Log Analytics workspace                       │
│ ├── Unified querying across resources                            │
│ └── Better cost management                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Setting Up Application Insights (Portal Walkthrough)

```
Console → Application Insights → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE APPLICATION INSIGHTS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-myapp-prod ▼]                                  │
│                                                                       │
│ Name: [appi-myapp-prod]                                            │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Resource Mode: [Workspace-based ▼] (recommended)                 │
│ Log Analytics Workspace: [law-myapp-prod ▼]                       │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation, you'll get:                                         │
│ ├── Connection String: InstrumentationKey=xxx;IngestionEndpoint=…│
│ │   ⚡ Use this in your application code/config                │
│ └── Instrumentation Key: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   │
│     ⚡ Legacy, use Connection String instead                    │
│                                                                       │
│ Enable on existing App Service (codeless):                         │
│ App Service → Settings → Application Insights → [Turn on]      │
│ ⚡ No code changes needed! Just enable in portal.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Auto-Instrumentation vs SDK

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTRUMENTATION OPTIONS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Option 1: Auto-instrumentation (codeless) ← Easiest              │
│ ├── Enable in portal (App Service, AKS, VMs)                    │
│ ├── No code changes needed                                       │
│ ├── Collects: requests, dependencies, exceptions, performance   │
│ ├── Supported: .NET, Java, Node.js, Python                      │
│ └── Limitation: No custom events/metrics                        │
│                                                                       │
│ Option 2: SDK (add code) ← More control                          │
│ ├── Install NuGet/npm/pip package                                │
│ ├── Add connection string to config                              │
│ ├── Track custom events, metrics, dependencies                  │
│ └── Full control over what's captured                            │
│                                                                       │
│ Node.js example:                                                    │
│ npm install applicationinsights                                    │
│                                                                       │
│ const appInsights = require('applicationinsights');               │
│ appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)│
│   .setAutoCollectRequests(true)                                   │
│   .setAutoCollectPerformance(true)                                │
│   .setAutoCollectExceptions(true)                                 │
│   .setAutoCollectDependencies(true)                               │
│   .start();                                                        │
│                                                                       │
│ // Custom event                                                     │
│ appInsights.defaultClient.trackEvent({                             │
│   name: "OrderPlaced",                                             │
│   properties: { orderId: "12345", amount: "99.99" }              │
│ });                                                                  │
│                                                                       │
│ .NET example:                                                       │
│ builder.Services.AddApplicationInsightsTelemetry();              │
│ // That's it! Auto-collects everything.                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Key Features

```
┌─────────────────────────────────────────────────────────────────────┐
│           APPLICATION INSIGHTS FEATURES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Live Metrics (real-time stream)                                  │
│    App Insights → Live Metrics                                    │
│    ├── Real-time request rate, response time, failure rate       │
│    ├── Server CPU and memory                                     │
│    └── Live stream of incoming requests and failures             │
│                                                                       │
│ 2. Application Map (visual topology)                               │
│    App Insights → Application Map                                 │
│    ├── Shows all components (web app, API, database, cache)     │
│    ├── Lines show dependencies and call rates                    │
│    ├── Red circles = failures                                     │
│    └── Click any node for details                                │
│    ┌─────────┐      ┌─────────┐      ┌─────────┐             │
│    │ Frontend │─────→│ API     │─────→│ SQL DB  │             │
│    │ 1.2k/min│      │ 800/min │      │ 500/min │             │
│    │ 0.1% err│      │ 0.5% err│      │ 0% err  │             │
│    └─────────┘      └────┬────┘      └─────────┘             │
│                          │                                       │
│                     ┌────▼────┐                                  │
│                     │ Redis   │                                  │
│                     │ 1.5k/min│                                  │
│                     └─────────┘                                  │
│                                                                       │
│ 3. Failures (error analysis)                                       │
│    App Insights → Failures                                        │
│    ├── Failed requests by operation and exception type           │
│    ├── Drill into stack traces                                   │
│    └── See exact request details that caused failure             │
│                                                                       │
│ 4. Performance                                                      │
│    App Insights → Performance                                     │
│    ├── Response time distribution                                │
│    ├── Slowest operations                                        │
│    ├── Dependency performance (SQL, HTTP, Redis)                │
│    └── Server response time percentiles (p50, p95, p99)        │
│                                                                       │
│ 5. Users, Sessions, Events                                         │
│    ├── How many users? Where from?                               │
│    ├── Session duration, page views                              │
│    └── Custom events (business metrics)                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Distributed Tracing

```
┌─────────────────────────────────────────────────────────────────────┐
│           DISTRIBUTED TRACING                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: A user request hits multiple services.                     │
│ Frontend → API Gateway → Order Service → Payment API → DB     │
│ How do you trace ONE request across ALL services?               │
│                                                                       │
│ Solution: Each request gets an Operation ID that follows it      │
│ through every service.                                              │
│                                                                       │
│ App Insights → Transaction search → Click any request           │
│                                                                       │
│ End-to-end transaction:                                              │
│ ┌──────────────────────────────────────────────────────────┐    │
│ │ Request: POST /api/orders  (200ms total)                  │    │
│ │ Operation ID: abc-123-xyz                                   │    │
│ │                                                              │    │
│ │ ├── Frontend (50ms)                                        │    │
│ │ │   └── HTTP GET /api/orders                              │    │
│ │ ├── Order Service (80ms)                                  │    │
│ │ │   ├── SQL: INSERT INTO Orders (30ms) ✅                │    │
│ │ │   └── HTTP: POST payment-api.com/charge (40ms) ✅     │    │
│ │ └── Email Service (70ms)                                  │    │
│ │     └── SendGrid API call (60ms) ✅                      │    │
│ └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│ ⚡ You can see exactly WHERE the time was spent!                 │
│ ⚡ If any step failed, you see the error details.               │
│                                                                       │
│ KQL query for tracing:                                              │
│ union AppRequests, AppDependencies, AppExceptions                 │
│ | where OperationId == "abc-123-xyz"                              │
│ | sort by TimeGenerated asc                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Availability Tests

```
┌─────────────────────────────────────────────────────────────────────┐
│           AVAILABILITY TESTS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ App Insights → Availability → [+ Add test]                       │
│                                                                       │
│ Test types:                                                          │
│                                                                       │
│ 1. URL ping test (simple):                                          │
│    URL: [https://myapp.azurewebsites.net/health]                 │
│    Frequency: [Every 5 minutes ▼]                                │
│    Test locations: ☑ Central India ☑ East US ☑ West Europe     │
│    Success criteria:                                                │
│      HTTP response: [200]                                         │
│      Content match: ["healthy"]                                   │
│    Timeout: [120] seconds                                         │
│    Alerts: [ag-ops-team]                                          │
│                                                                       │
│ 2. Standard test (more options):                                    │
│    HTTP method: GET/POST/PUT/HEAD                                 │
│    Headers, body, SSL validation                                  │
│    Follow redirects, parse dependent requests                    │
│                                                                       │
│ 3. Custom TrackAvailability test (code-based):                     │
│    Write custom test in Azure Function                            │
│    For complex multi-step scenarios                               │
│                                                                       │
│ ⚡ Tests run from 5+ Azure regions worldwide.                    │
│ ⚡ If your app is down from any region, you get alerted.        │
│                                                                       │
│ Dashboard shows:                                                    │
│ ├── Availability % by location (99.9%, 100%, etc.)              │
│ ├── Response time trends                                         │
│ └── Failure details (timeout, HTTP error, content mismatch)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Smart Detection & Alerts

```
Smart Detection (automatic, AI-powered):
├── Failure anomalies → Unusual spike in failed requests
├── Performance anomalies → Sudden increase in response time
├── Memory leak detection → Gradual memory increase
├── Dependency slowness → SQL/HTTP calls getting slower
├── Trace severity → Spike in error-level logs
└── ⚡ No configuration needed! Works automatically.

Manual alerts (configure yourself):
├── Response time > 2 seconds for 5 minutes
├── Failure rate > 5% for 10 minutes
├── Exception count > 50 in 15 minutes
└── Custom event count (e.g., failed payments > 10)
```

---

## Part 8: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_application_insights" "main" {
  name                = "appi-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  workspace_id        = azurerm_log_analytics_workspace.main.id
  application_type    = "web"
}

output "app_insights_connection_string" {
  value     = azurerm_application_insights.main.connection_string
  sensitive = true
}

# Availability test
resource "azurerm_application_insights_standard_web_test" "health" {
  name                    = "health-check"
  resource_group_name     = azurerm_resource_group.main.name
  location                = azurerm_resource_group.main.location
  application_insights_id = azurerm_application_insights.main.id
  frequency               = 300 # seconds
  timeout                 = 120
  enabled                 = true
  geo_locations           = ["us-tx-sn1-azr", "apac-sg-sin-azr"]

  request {
    url = "https://myapp.azurewebsites.net/health"
  }
}
```

### Bicep

```bicep
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-myapp-prod'
  location: resourceGroup().location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalyticsWorkspace.id
  }
}

output connectionString string = appInsights.properties.ConnectionString
```

---

## Part 9: az CLI Reference

```bash
# Create Application Insights
az monitor app-insights component create \
  --app appi-myapp-prod \
  --location centralindia \
  --resource-group rg-myapp-prod \
  --workspace law-myapp-prod \
  --application-type web

# Show connection string
az monitor app-insights component show \
  --app appi-myapp-prod \
  --resource-group rg-myapp-prod \
  --query connectionString

# Query Application Insights data
az monitor app-insights query \
  --app appi-myapp-prod \
  --resource-group rg-myapp-prod \
  --analytics-query "requests | summarize count() by resultCode | sort by count_ desc"

# List Application Insights resources
az monitor app-insights component show \
  --resource-group rg-myapp-prod

# Delete
az monitor app-insights component delete \
  --app appi-myapp-prod \
  --resource-group rg-myapp-prod
```

---

## Real-World Patterns

### Pattern 1: APM for Microservices

```
┌─────────────────────────────────────────────────┐
│  Distributed Tracing across Services            │
├─────────────────────────────────────────────────┤
│                                                 │
│  User Request ─→ API Gateway                    │
│       │          (trace-id: abc-123)             │
│       ▼                                         │
│  Order Service ─→ Payment Service               │
│       │                   │                      │
│       ▼                   ▼                      │
│  Inventory Service    Notification Service      │
│                                                 │
│  All services share same App Insights resource  │
│  Application Map shows dependencies visually    │
│  End-to-end transaction search by trace-id      │
│                                                 │
│  Smart Detection: Auto-alerts on anomalies      │
│  (failure rate spikes, slow responses)          │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Application Insights = APM (Application Performance Monitoring)
Monitors: Requests, dependencies, exceptions, performance, users

Setup options:
  Auto-instrumentation: Enable in portal (no code changes)
  SDK: Install package + connection string (more control)

Key features:
  Live Metrics: Real-time request/failure stream
  Application Map: Visual topology of your app
  Failures: Error analysis with stack traces
  Performance: Response times, slow operations
  Distributed Tracing: Track request across microservices
  Availability Tests: Ping your app from worldwide locations

Smart Detection: AI-powered anomaly detection (automatic)
Data stored in: Log Analytics workspace (query with KQL)

Connection String: Use this in your app config
Tables: AppRequests, AppDependencies, AppExceptions, AppTraces
```

---

## What's Next?

Next chapter: [Chapter 41: Azure Service Health & Advisor](41-service-health-advisor.md) — Service health alerts, planned maintenance notifications, and optimization recommendations.
