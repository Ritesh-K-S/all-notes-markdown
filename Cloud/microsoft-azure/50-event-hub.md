# Chapter 50: Azure Event Hub

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Event Hub Fundamentals](#part-1-event-hub-fundamentals)
- [Part 2: Creating an Event Hub (Portal Walkthrough)](#part-2-creating-an-event-hub-portal-walkthrough)
- [Part 3: Partitions & Consumer Groups](#part-3-partitions--consumer-groups)
- [Part 4: Capture (Auto-Archive)](#part-4-capture-auto-archive)
- [Part 5: Schema Registry](#part-5-schema-registry)
- [Part 6: Terraform & az CLI Reference](#part-6-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Event Hub is a big data streaming platform that can receive and process millions of events per second. It's designed for telemetry, logging, IoT data, and clickstream analytics — scenarios where you have a firehose of data that needs to be ingested and processed.

```
What you'll learn:
├── Event Hub Fundamentals
│   ├── What is event streaming (high-volume data ingestion)
│   ├── Event Hub vs Service Bus vs Event Grid
│   └── Tiers & throughput units
├── Creating an Event Hub (Portal)
├── Partitions & Consumer Groups
├── Capture (auto-archive to Storage/Data Lake)
├── Schema Registry (Avro schemas)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Event Hub Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT HUB OVERVIEW                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Event Hub = Data streaming pipeline (like Apache Kafka)            │
│                                                                       │
│ Use cases:                                                           │
│ ├── Application telemetry (millions of log events/sec)          │
│ ├── IoT device data (sensors, GPS, temperature)                 │
│ ├── Clickstream analytics (website user behavior)               │
│ ├── Financial transaction streaming                              │
│ └── Live dashboards and real-time analytics                     │
│                                                                       │
│ How it works:                                                        │
│ Producers (senders)                                                 │
│ ├── App servers, IoT devices, log agents                        │
│ ▼                                                                  │
│ ┌──────────────────────────────────────────┐                      │
│ │ Event Hub                                  │                      │
│ │ Partition 0: [e1][e2][e3][e4][e5]...     │                      │
│ │ Partition 1: [e6][e7][e8][e9]...         │                      │
│ │ Partition 2: [e10][e11][e12]...          │                      │
│ │ Partition 3: [e13][e14][e15]...          │                      │
│ │                                            │                      │
│ │ Retention: 1-90 days                     │                      │
│ └────────────────┬─────────────────────────┘                      │
│                  ▼                                                  │
│ Consumers (readers)                                                 │
│ ├── Azure Stream Analytics                                       │
│ ├── Azure Functions                                               │
│ ├── Apache Spark / Databricks                                    │
│ └── Custom consumer apps                                         │
│                                                                       │
│ Tiers:                                                               │
│ ├── Basic: 1 consumer group, 1-day retention                   │
│ ├── Standard: 20 consumer groups, 7-day retention (~$11/TU/mo) │
│ ├── Premium: 100 consumer groups, 90-day retention             │
│ └── Dedicated: Single-tenant cluster, highest throughput       │
│                                                                       │
│ Throughput Units (TU): 1 TU = 1 MB/s ingress, 2 MB/s egress   │
│ Auto-inflate: Auto-scale TUs up to a maximum                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating an Event Hub (Full Portal Walkthrough)

```
Console → Event Hubs → Create (Namespace first, then Event Hub)

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 1: CREATE EVENT HUB NAMESPACE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Basics tab ──                                                    │
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-streaming ▼]                                   │
│                                                                       │
│ Namespace name: [ehns-myapp-prod]                                  │
│ ⚡ Globally unique. Creates endpoint:                              │
│   ehns-myapp-prod.servicebus.windows.net                           │
│                                                                       │
│ Location: [Central India ▼]                                        │
│ ⚡ Same region as your producers and consumers for lowest latency│
│                                                                       │
│ Pricing tier: [Standard ▼]                                        │
│ ┌──────────────┬───────────────────────────────────────────────┐  │
│ │ Tier         │ Features                                      │  │
│ ├──────────────┼───────────────────────────────────────────────┤  │
│ │ Basic        │ 1 consumer group, 1-day retention, $0.015/hr │  │
│ │ Standard     │ 20 consumer groups, 7-day, Kafka, $0.03/hr   │  │
│ │ Premium      │ 100 groups, 90-day, zone redundancy          │  │
│ │ Dedicated    │ Single-tenant cluster, highest throughput     │  │
│ └──────────────┴───────────────────────────────────────────────┘  │
│                                                                       │
│ Throughput Units: [1] (1-40)                                       │
│ ⚡ 1 TU = 1 MB/s ingress + 2 MB/s egress                        │
│ ⚡ Start with 1 TU and increase as needed.                       │
│                                                                       │
│ ☑ Enable Auto-Inflate                                              │
│ Maximum Throughput Units: [10]                                     │
│ ⚡ Auto-scales TUs up to this limit during traffic spikes.       │
│                                                                       │
│ ── Networking tab ──                                                │
│                                                                       │
│ Connectivity method:                                                 │
│ ● Public endpoint (all networks)                                  │
│ ○ Public endpoint (selected networks + IP rules)                 │
│ ○ Private endpoint                                                │
│ ⚡ For production, use private endpoints or selected networks.   │
│                                                                       │
│ ── Tags tab ──                                                      │
│ environment: production                                              │
│ team: data-platform                                                  │
│                                                                       │
│ [Review + Create] → [Create]                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 2: CREATE EVENT HUB INSIDE NAMESPACE                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Namespace → Event Hubs → [+ Event Hub]                           │
│                                                                       │
│ Name: [telemetry-stream]                                           │
│ ⚡ Unique within the namespace                                    │
│                                                                       │
│ Partition count: [4] (2-32)                                       │
│ ⚡ CANNOT be changed after creation!                              │
│ ⚡ More partitions = more parallel consumers = higher throughput │
│ ⚡ Recommendation: 4-8 for most workloads, 32 for high-volume   │
│                                                                       │
│ Message retention: [7] days                                        │
│ ├── Basic: 1 day only                                            │
│ ├── Standard: 1-7 days                                           │
│ └── Premium: 1-90 days                                           │
│                                                                       │
│ Capture: [On/Off]                                                  │
│ ⚡ Auto-archive to Storage or Data Lake (see Part 4)             │
│                                                                       │
│ Cleanup policy: [Delete ▼]                                       │
│ ├── Delete: Remove events after retention period (default)      │
│ └── Compact: Keep latest event per key (like Kafka compaction) │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Partitions & Consumer Groups

```
Partitions:
├── Event Hub splits data into partitions (like lanes on a highway)
├── Each partition is an independent, ordered sequence of events
├── More partitions = more parallel consumers = higher throughput
├── Events with same partition key go to same partition (ordering!)
├── Cannot change partition count after creation
└── Recommendation: Start with 4-8 partitions for most workloads

Consumer Groups:
├── A "view" of the event hub for a group of consumers
├── Each consumer group tracks its own position (offset)
├── Multiple groups can read the same data independently
├── Default: $Default consumer group (always exists)
│
├── Example:
│   Consumer Group: "analytics" → Spark job reads all events
│   Consumer Group: "alerts" → Function reads for anomalies
│   Consumer Group: "archive" → Data Factory archives to Data Lake
│   ⚡ Each group reads ALL events independently!
│
└── Standard: Up to 20 consumer groups per Event Hub
```

---

## Part 4: Capture (Auto-Archive)

```
Event Hub → Capture → On

┌─────────────────────────────────────────────────────────────────────┐
│           CAPTURE (Auto-Archive)                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automatically saves events to Storage or Data Lake (Avro format) │
│                                                                       │
│ Capture provider: ● Azure Blob Storage  ○ Azure Data Lake Store│
│ Storage account: [starchive ▼]                                    │
│ Container: [event-archive]                                         │
│                                                                       │
│ Time window: [5] minutes (1-15 min)                                │
│ Size window: [300] MB (10-500 MB)                                 │
│ ⚡ Whichever limit is hit first triggers a capture file.         │
│                                                                       │
│ Output format: Avro files                                          │
│ Path: {Namespace}/{EventHub}/{Partition}/{Year}/{Month}/{Day}/   │
│                                                                       │
│ ⚡ Zero-code data archival!                                      │
│ ⚡ No separate consumer needed for archiving.                   │
│ ⚡ Query archived data with Azure Synapse or Databricks.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Schema Registry

```
Schema Registry:
├── Store and manage Avro schemas centrally
├── Producers and consumers agree on data format
├── Schema evolution (add fields without breaking consumers)
├── Namespace → Schema Registry → [+ Schema Group]
└── Use with Kafka client compatibility (Event Hub supports Kafka protocol!)

Event Hub for Apache Kafka:
├── Event Hub is Kafka-compatible!
├── Use existing Kafka producers/consumers with Event Hub
├── Just change the bootstrap server to Event Hub endpoint
├── No need to manage your own Kafka cluster
└── Connection: ehns-myapp-prod.servicebus.windows.net:9093
```

---

## Part 6: Managing & Deleting (Portal)

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING EVENT HUB                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Namespace → Overview:                                               │
│ ├── Requests: Total incoming/outgoing requests                   │
│ ├── Throttled requests: Requests exceeding TU limits            │
│ ├── Incoming/Outgoing messages: Event count                     │
│ ├── Incoming/Outgoing bytes: Data volume                        │
│ └── ⚡ Watch throttled requests — scale TUs if > 0!             │
│                                                                       │
│ Namespace → Scale:                                                  │
│ ├── Throughput Units: Increase/decrease (1-40)                  │
│ ├── Auto-inflate: Enable/disable and set max TUs               │
│ └── ⚡ Scale up before expected traffic spikes                  │
│                                                                       │
│ Namespace → Shared access policies:                                 │
│ ├── RootManageSharedAccessKey (full access — default)          │
│ ├── Create per-hub policies for specific access                 │
│ ├── [+ Add] → Name, Manage/Send/Listen permissions            │
│ └── Get connection strings for producers and consumers         │
│                                                                       │
│ Event Hub → Consumer groups:                                       │
│ ├── View all consumer groups and their partition ownership      │
│ ├── [$Default] is always present                                │
│ └── [+ Consumer group] → Name → Create                       │
│                                                                       │
│ Event Hub → Process data:                                          │
│ ├── Test with "Process data" feature (portal event explorer)    │
│ ├── View real-time events flowing through the hub              │
│ └── Great for debugging and verifying data flow                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DELETING EVENT HUB / NAMESPACE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Delete single Event Hub:                                            │
│ Namespace → Event Hubs → [hub-name] → [Delete]                  │
│ ⚡ Deletes the hub and all events. Consumer groups also deleted. │
│                                                                       │
│ Delete entire Namespace:                                            │
│ Namespace → Overview → [Delete]                                  │
│ ⚡ Deletes ALL event hubs inside the namespace!                  │
│ ⚡ Type namespace name to confirm.                               │
│                                                                       │
│ CLI:                                                                 │
│   az eventhubs eventhub delete \                                   │
│     --name telemetry-stream \                                      │
│     --namespace-name ehns-myapp-prod \                             │
│     --resource-group rg-streaming                                  │
│                                                                       │
│   az eventhubs namespace delete \                                   │
│     --name ehns-myapp-prod \                                       │
│     --resource-group rg-streaming                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_eventhub_namespace" "main" {
  name                = "ehns-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
  capacity            = 1
  auto_inflate_enabled   = true
  maximum_throughput_units = 10

  tags = {
    environment = "production"
  }
}

resource "azurerm_eventhub" "telemetry" {
  name                = "telemetry-stream"
  namespace_name      = azurerm_eventhub_namespace.main.name
  resource_group_name = azurerm_resource_group.main.name
  partition_count     = 4
  message_retention   = 7

  capture_description {
    enabled             = true
    encoding            = "Avro"
    interval_in_seconds = 300
    size_limit_in_bytes = 314572800

    destination {
      name                = "EventHubArchive.AzureBlockBlob"
      archive_name_format = "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
      blob_container_name = "event-archive"
      storage_account_id  = azurerm_storage_account.archive.id
    }
  }
}

resource "azurerm_eventhub_consumer_group" "analytics" {
  name                = "analytics"
  namespace_name      = azurerm_eventhub_namespace.main.name
  eventhub_name       = azurerm_eventhub.telemetry.name
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_eventhub_consumer_group" "alerts" {
  name                = "alerts"
  namespace_name      = azurerm_eventhub_namespace.main.name
  eventhub_name       = azurerm_eventhub.telemetry.name
  resource_group_name = azurerm_resource_group.main.name
}
```

### Bicep

```bicep
// Event Hub Namespace
resource namespace 'Microsoft.EventHub/namespaces@2024-01-01' = {
  name: 'ehns-myapp-prod'
  location: resourceGroup().location
  sku: {
    name: 'Standard'
    tier: 'Standard'
    capacity: 1
  }
  properties: {
    isAutoInflateEnabled: true
    maximumThroughputUnits: 10
  }
  tags: {
    environment: 'production'
  }
}

// Event Hub
resource eventHub 'Microsoft.EventHub/namespaces/eventhubs@2024-01-01' = {
  parent: namespace
  name: 'telemetry-stream'
  properties: {
    partitionCount: 4
    messageRetentionInDays: 7
    captureDescription: {
      enabled: true
      encoding: 'Avro'
      intervalInSeconds: 300
      sizeLimitInBytes: 314572800
      destination: {
        name: 'EventHubArchive.AzureBlockBlob'
        properties: {
          storageAccountResourceId: storageAccount.id
          blobContainer: 'event-archive'
          archiveNameFormat: '{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}'
        }
      }
    }
  }
}

// Consumer Groups
resource analyticsGroup 'Microsoft.EventHub/namespaces/eventhubs/consumergroups@2024-01-01' = {
  parent: eventHub
  name: 'analytics'
}

resource alertsGroup 'Microsoft.EventHub/namespaces/eventhubs/consumergroups@2024-01-01' = {
  parent: eventHub
  name: 'alerts'
}

// Authorization Rule (Send-only for producers)
resource sendRule 'Microsoft.EventHub/namespaces/eventhubs/authorizationRules@2024-01-01' = {
  parent: eventHub
  name: 'producer-key'
  properties: {
    rights: [
      'Send'
    ]
  }
}

// Output connection strings
output namespaceEndpoint string = namespace.properties.serviceBusEndpoint
output sendConnectionString string = listKeys(sendRule.id, '2024-01-01').primaryConnectionString
```

---

## Part 8: az CLI Reference

```bash
# ─── NAMESPACE ───

# Create namespace
az eventhubs namespace create \
  --name ehns-myapp-prod \
  --resource-group rg-streaming \
  --location centralindia \
  --sku Standard --capacity 1 \
  --enable-auto-inflate --maximum-throughput-units 10

# List namespaces
az eventhubs namespace list --resource-group rg-streaming --output table

# Show namespace details
az eventhubs namespace show \
  --name ehns-myapp-prod \
  --resource-group rg-streaming

# Update namespace (scale throughput units)
az eventhubs namespace update \
  --name ehns-myapp-prod \
  --resource-group rg-streaming \
  --capacity 5

# ─── EVENT HUB ───

# Create event hub
az eventhubs eventhub create \
  --name telemetry-stream \
  --namespace-name ehns-myapp-prod \
  --resource-group rg-streaming \
  --partition-count 4 --message-retention 7

# List event hubs in namespace
az eventhubs eventhub list \
  --namespace-name ehns-myapp-prod \
  --resource-group rg-streaming --output table

# Show event hub details
az eventhubs eventhub show \
  --name telemetry-stream \
  --namespace-name ehns-myapp-prod \
  --resource-group rg-streaming

# Update event hub (change retention)
az eventhubs eventhub update \
  --name telemetry-stream \
  --namespace-name ehns-myapp-prod \
  --resource-group rg-streaming \
  --message-retention 3

# ─── CONSUMER GROUPS ───

# Create consumer group
az eventhubs eventhub consumer-group create \
  --name analytics \
  --namespace-name ehns-myapp-prod \
  --eventhub-name telemetry-stream \
  --resource-group rg-streaming

# List consumer groups
az eventhubs eventhub consumer-group list \
  --namespace-name ehns-myapp-prod \
  --eventhub-name telemetry-stream \
  --resource-group rg-streaming --output table

# Delete consumer group
az eventhubs eventhub consumer-group delete \
  --name analytics \
  --namespace-name ehns-myapp-prod \
  --eventhub-name telemetry-stream \
  --resource-group rg-streaming

# ─── ACCESS & CONNECTION STRINGS ───

# Get namespace connection string
az eventhubs namespace authorization-rule keys list \
  --namespace-name ehns-myapp-prod \
  --resource-group rg-streaming \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv

# Create hub-level access policy (Send only for producers)
az eventhubs eventhub authorization-rule create \
  --name producer-key \
  --namespace-name ehns-myapp-prod \
  --eventhub-name telemetry-stream \
  --resource-group rg-streaming \
  --rights Send

# Get hub-level connection string
az eventhubs eventhub authorization-rule keys list \
  --name producer-key \
  --namespace-name ehns-myapp-prod \
  --eventhub-name telemetry-stream \
  --resource-group rg-streaming \
  --query primaryConnectionString -o tsv

# ─── DELETE ───

# Delete event hub
az eventhubs eventhub delete \
  --name telemetry-stream \
  --namespace-name ehns-myapp-prod \
  --resource-group rg-streaming

# Delete namespace (and all event hubs inside)
az eventhubs namespace delete \
  --name ehns-myapp-prod \
  --resource-group rg-streaming
```

---

## Part 9: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: IOT TELEMETRY PIPELINE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 10,000 IoT sensors → Event Hub → Multiple consumers             │
│                                                                       │
│ ┌──────────┐    ┌─────────────────┐    ┌──────────────────────┐  │
│ │ Sensors  │──→ │ Event Hub       │──→ │ CG: "realtime"       │  │
│ │ (temp,   │    │ 32 partitions   │    │ → Stream Analytics   │  │
│ │ humidity,│    │ 10 TU           │    │ → Alert if temp > 80 │  │
│ │ pressure)│    │                 │──→ │ CG: "analytics"      │  │
│ └──────────┘    │                 │    │ → Databricks hourly  │  │
│                 │                 │──→ │ CG: "archive"        │  │
│                 │                 │    │ → Capture → ADLS     │  │
│                 └─────────────────┘    └──────────────────────┘  │
│                                                                       │
│ Each consumer group processes all events independently!           │
│ Real-time alerts + batch analytics + long-term archive.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: CLICKSTREAM ANALYTICS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Website/App (millions of users)                                     │
│ → Every click, page view, scroll → Event Hub                    │
│ → Stream Analytics (real-time): Dashboard showing live users     │
│ → Capture (archive): All events → Data Lake → ML training      │
│                                                                       │
│ Partition key: user_id (all events for one user → same partition)│
│ Result: Ordered event stream per user for session analysis       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: APPLICATION LOGGING AT SCALE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 100 microservices → Event Hub → Central logging                  │
│                                                                       │
│ Each service sends structured log events:                          │
│ { service: "payment-api", level: "error", message: "..." }       │
│                                                                       │
│ CG: "monitoring" → Azure Monitor / Log Analytics                 │
│ CG: "compliance" → Archive to immutable storage (audit logs)    │
│ CG: "debugging" → Real-time search for specific errors          │
│                                                                       │
│ Why Event Hub (not Log Analytics directly)?                        │
│ ├── Handles millions of events/sec (Log Analytics has limits)   │
│ ├── Multiple consumers process the same data                    │
│ └── Buffer during ingestion spikes (Log Analytics can throttle) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT HUB QUICK REFERENCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Big data streaming platform (millions events/sec)            │
│ Like: Apache Kafka, but fully managed (Kafka-compatible!)         │
│                                                                       │
│ Hierarchy: Namespace → Event Hub → Partitions → Consumer Groups │
│                                                                       │
│ Partitions: Parallel processing lanes (2-32, set at creation)     │
│ Consumer Groups: Independent readers of the same data             │
│ TU: 1 Throughput Unit = 1 MB/s in, 2 MB/s out                   │
│                                                                       │
│ Capture: Auto-archive to Storage/Data Lake (Avro format)          │
│ Kafka: Event Hub is Kafka-compatible (use Kafka clients!)         │
│ Schema Registry: Manage Avro schemas centrally                    │
│                                                                       │
│ Tiers:                                                               │
│ ├── Basic: 1 consumer group, 1-day retention                    │
│ ├── Standard: 20 groups, 7-day, Kafka ($11/TU/month)           │
│ ├── Premium: 100 groups, 90-day, zone redundancy               │
│ └── Dedicated: Single-tenant cluster, highest throughput        │
│                                                                       │
│ vs Service Bus: Event Hub = streaming; Service Bus = messaging   │
│ vs Event Grid: Event Hub = high-volume data; Event Grid = events│
│                                                                       │
│ ⚡ AWS equivalent: Amazon Kinesis Data Streams                    │
│ ⚡ GCP equivalent: Cloud Pub/Sub (streaming mode)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Next chapter: [Chapter 51: Azure Queue Storage](51-queue-storage.md) — Simple, low-cost message queuing for background processing. Learn when to use the simplest and cheapest messaging option in Azure.
