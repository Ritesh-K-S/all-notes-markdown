# Chapter 59: Azure Stream Analytics

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Stream Analytics Fundamentals](#part-1-stream-analytics-fundamentals)
- [Part 2: Creating a Stream Analytics Job (Portal Walkthrough)](#part-2-creating-a-stream-analytics-job-portal-walkthrough)
- [Part 3: Query Language](#part-3-query-language)
- [Part 4: Windowing Functions](#part-4-windowing-functions)
- [Part 5: Terraform & az CLI Reference](#part-5-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Stream Analytics is a real-time analytics service that processes streaming data using SQL-like queries. Data flows in from Event Hub, IoT Hub, or Blob Storage, gets processed by your query, and results go to outputs like SQL Database, Power BI, or Blob Storage.

```
What you'll learn:
├── Stream Analytics Fundamentals (real-time processing)
├── Creating a Job (Portal)
├── Query Language (SQL-like)
├── Windowing Functions (tumbling, hopping, sliding, session)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Stream Analytics Fundamentals

```
Stream Analytics = Real-time SQL on streaming data

How it works:
  Input (streaming)           Query              Output
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Event Hub    │ →  │ SELECT ...   │ →  │ SQL Database │
  │ IoT Hub      │    │ FROM Input   │    │ Power BI     │
  │ Blob Storage │    │ WHERE ...    │    │ Blob Storage │
  └──────────────┘    │ GROUP BY ... │    │ Cosmos DB    │
                      │ WINDOW ...   │    │ Event Hub    │
                      └──────────────┘    │ Service Bus  │
                                          │ Azure Function│
                                          └──────────────┘

Use cases:
├── IoT: Process sensor data in real-time (temperature alerts)
├── Fraud detection: Flag suspicious transactions immediately
├── Live dashboards: Real-time metrics to Power BI
├── Anomaly detection: Built-in ML function
├── Log processing: Filter and aggregate log streams
└── Clickstream: Real-time user behavior analytics

Pricing:
├── Streaming Units (SU): ~$0.11/SU/hour
├── 1 SU = Minimum (dev/test)
├── 3 SU = Typical small workload
├── Scale up for higher throughput
└── ~$80/month for 1 SU running 24/7
```

---

## Part 2: Creating a Stream Analytics Job (Portal Walkthrough)

```
Console → Stream Analytics jobs → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE STREAM ANALYTICS JOB                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-streaming ▼]                                   │
│ Name: [asa-iot-processor]                                          │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Hosting environment:                                                 │
│ ● Cloud                                                             │
│ ○ Edge (run on IoT Edge devices)                                  │
│                                                                       │
│ Streaming units: [3]                                                │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation, configure 3 things:                                │
│                                                                       │
│ 1. INPUT:                                                            │
│    Job → Inputs → [+ Add stream input]                            │
│    Type: [Event Hub ▼]                                             │
│    Alias: [sensor-data]                                             │
│    Event Hub namespace: [ehns-iot ▼]                               │
│    Event Hub: [sensors ▼]                                          │
│    Consumer group: [stream-analytics]                              │
│                                                                       │
│ 2. OUTPUT:                                                           │
│    Job → Outputs → [+ Add]                                        │
│    Type: [SQL Database ▼]                                          │
│    Alias: [sql-output]                                              │
│    Database: [iotdb ▼]                                              │
│    Table: [SensorAlerts]                                            │
│                                                                       │
│ 3. QUERY:                                                            │
│    Job → Query                                                      │
│    (see Part 3)                                                     │
│                                                                       │
│ Then: [▶ Start] the job                                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Query Language

```
SQL-like syntax with streaming extensions:

-- Basic filter
SELECT deviceId, temperature, humidity, EventProcessedUtcTime
INTO [sql-output]
FROM [sensor-data]
WHERE temperature > 80

-- Aggregation with time window (5-minute averages)
SELECT
    deviceId,
    AVG(temperature) AS avgTemp,
    MAX(temperature) AS maxTemp,
    COUNT(*) AS readingCount,
    System.Timestamp() AS windowEnd
INTO [sql-output]
FROM [sensor-data]
TIMESTAMP BY eventTime
GROUP BY deviceId, TumblingWindow(minute, 5)

-- Reference data JOIN (enrich stream with static data)
-- Reference input: [device-info] from Blob Storage
SELECT
    s.deviceId,
    d.deviceName,
    d.location,
    s.temperature
INTO [sql-output]
FROM [sensor-data] s
JOIN [device-info] d ON s.deviceId = d.deviceId
WHERE s.temperature > 80

-- Anomaly detection (built-in ML)
SELECT
    deviceId,
    temperature,
    AnomalyDetection_SpikeAndDip(temperature, 95, 120) OVER (
        PARTITION BY deviceId
        LIMIT DURATION(minute, 10)
    ) AS anomalyResult
INTO [anomaly-output]
FROM [sensor-data]
```

---

## Part 4: Windowing Functions

```
Windows = Group events by time periods

1. Tumbling Window (fixed, non-overlapping):
   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
   │ 5m  │ │ 5m  │ │ 5m  │ │ 5m  │
   └─────┘ └─────┘ └─────┘ └─────┘
   No gaps, no overlap. Each event belongs to exactly 1 window.

   GROUP BY TumblingWindow(minute, 5)

2. Hopping Window (fixed, overlapping):
   ┌─────────┐
   │   10m   │
   └─────────┘
      ┌─────────┐
      │   10m   │
      └─────────┘
         ┌─────────┐
         │   10m   │
         └─────────┘
   Window size 10 min, hop 5 min → 50% overlap

   GROUP BY HoppingWindow(minute, 10, 5)

3. Sliding Window (event-triggered):
   Only fires when events enter/exit the window.
   "Alert if >5 events in any 10-min period"

   GROUP BY SlidingWindow(minute, 10)
   HAVING COUNT(*) > 5

4. Session Window (gap-based):
   ┌──────────┐    ┌───────┐       ┌──────────────┐
   │ session1 │gap │ ses2  │  gap  │   session3   │
   └──────────┘    └───────┘       └──────────────┘
   Groups events with gaps < timeout.

   GROUP BY SessionWindow(minute, 5, 30)
   -- 5 min timeout, 30 min max duration

5. Snapshot Window:
   Groups events that arrive at the same timestamp.

   GROUP BY System.Timestamp()
```

---

## Part 5: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_stream_analytics_job" "main" {
  name                                     = "asa-iot-processor"
  resource_group_name                      = azurerm_resource_group.main.name
  location                                 = azurerm_resource_group.main.location
  streaming_units                          = 3
  transformation_query                     = <<QUERY
    SELECT deviceId, AVG(temperature) as avgTemp
    INTO [sql-output]
    FROM [sensor-data]
    GROUP BY deviceId, TumblingWindow(minute, 5)
  QUERY
}

resource "azurerm_stream_analytics_stream_input_eventhub" "input" {
  name                         = "sensor-data"
  stream_analytics_job_name    = azurerm_stream_analytics_job.main.name
  resource_group_name          = azurerm_resource_group.main.name
  eventhub_name                = azurerm_eventhub.sensors.name
  eventhub_consumer_group_name = "$Default"
  servicebus_namespace         = azurerm_eventhub_namespace.main.name
  shared_access_policy_key     = azurerm_eventhub_namespace.main.default_primary_key
  shared_access_policy_name    = "RootManageSharedAccessKey"

  serialization {
    type     = "Json"
    encoding = "UTF8"
  }
}
```

### Bicep

```bicep
// Stream Analytics Job
resource streamJob 'Microsoft.StreamAnalytics/streamingjobs@2021-10-01-preview' = {
  name: 'asa-realtime-alerts'
  location: resourceGroup().location
  properties: {
    sku: { name: 'Standard' }
    compatibilityLevel: '1.2'
    eventsOutOfOrderPolicy: 'Adjust'
    eventsOutOfOrderMaxDelayInSeconds: 10
    transformation: {
      name: 'query'
      properties: {
        streamingUnits: 3
        query: 'SELECT * INTO [output] FROM [input] WHERE temperature > 80'
      }
    }
  }
}
```

### az CLI

```bash
# Create job
az stream-analytics job create \
  --name asa-iot-processor \
  --resource-group rg-streaming \
  --location centralindia \
  --streaming-units 3

# Start job
az stream-analytics job start \
  --name asa-iot-processor \
  --resource-group rg-streaming \
  --output-start-mode JobStartTime

# Stop job
az stream-analytics job stop \
  --name asa-iot-processor \
  --resource-group rg-streaming

# Show job status
az stream-analytics job show \
  --name asa-iot-processor \
  --resource-group rg-streaming -o table

# List jobs
az stream-analytics job list --resource-group rg-streaming -o table

# Scale streaming units
az stream-analytics job update \
  --name asa-iot-processor \
  --resource-group rg-streaming \
  --streaming-units 6

# Add Event Hub input
az stream-analytics input create \
  --job-name asa-iot-processor \
  --resource-group rg-streaming \
  --input-name sensor-data \
  --type Stream \
  --datasource @eventhub-input.json

# List inputs/outputs
az stream-analytics input list --job-name asa-iot-processor --resource-group rg-streaming -o table
az stream-analytics output list --job-name asa-iot-processor --resource-group rg-streaming -o table

# Delete job
az stream-analytics job delete \
  --name asa-iot-processor \
  --resource-group rg-streaming --yes
```

---

## Real-World Patterns

### Pattern 1: Real-Time IoT Alerting

```
┌─────────────────────────────────────────────────┐
│  IoT Temperature Monitoring Pipeline            │
├─────────────────────────────────────────────────┤
│                                                 │
│  IoT Sensors ─→ IoT Hub ─→ Stream Analytics     │
│  (1000 devices)          │                      │
│                     ┌────┴────┐                  │
│                     │ Query   │                  │
│                     │ AVG temp│                  │
│                     │ per 5min│                  │
│                     │ window  │                  │
│                     └────┬────┘                  │
│                ┌─────┼─────┐                     │
│                ▼         ▼                       │
│           Power BI   Service Bus                │
│           (dashboard) (alert if >80°C)           │
│                         │                       │
│                    Azure Function               │
│                    (send SMS alert)              │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Stream Analytics = Real-time SQL on streaming data

Architecture: Input → Query → Output
Inputs: Event Hub, IoT Hub, Blob Storage
Outputs: SQL DB, Power BI, Blob, Cosmos, Event Hub, Functions

Query: SQL-like with streaming extensions
Windows: Tumbling | Hopping | Sliding | Session | Snapshot

Streaming Units (SU): ~$0.11/SU/hour (1 SU minimum)
Built-in: Anomaly detection, reference data JOIN, JavaScript UDFs

Tumbling: Fixed, no overlap (most common)
Hopping: Fixed, with overlap
Sliding: Event-triggered
Session: Gap-based grouping
```

---

## What's Next?

Next chapter: [Chapter 60: Azure Machine Learning](60-azure-ml.md) — End-to-end machine learning platform for training, deploying, and managing models.
