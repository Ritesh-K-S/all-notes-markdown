# Chapter 49: Azure Event Grid

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Event Grid Fundamentals](#part-1-event-grid-fundamentals)
- [Part 2: Creating Event Subscriptions (Full Portal Walkthrough)](#part-2-creating-event-subscriptions-full-portal-walkthrough)
- [Part 3: Event Sources & Handlers](#part-3-event-sources--handlers)
- [Part 4: Custom Topics (Portal Walkthrough)](#part-4-custom-topics-portal-walkthrough)
- [Part 5: Filtering & Dead-Letter](#part-5-filtering--dead-letter)
- [Part 6: Managing & Deleting Event Subscriptions](#part-6-managing--deleting-event-subscriptions)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Event Grid is a fully managed event routing service. It reacts to events happening in Azure (blob uploaded, resource created, VM stopped) and routes them to handlers (Functions, Logic Apps, webhooks). Think of it as "when X happens, do Y."

**Simple analogy:** Imagine a smart doorbell system. When someone rings the bell (event), the system can send you a phone notification, record a video, turn on the porch light, or unlock the door — all automatically. Event Grid works the same way: when something happens in Azure, it notifies the services you choose.

```
What you'll learn:
├── Event Grid Fundamentals (event-driven architecture)
├── Creating Event Subscriptions (Full Portal Walkthrough)
├── Event Sources (Azure services that publish events)
├── Event Handlers (services that react to events)
├── Custom Topics (publish your own events)
├── Filtering & Dead-Letter
├── Managing & Deleting
├── Terraform, Bicep, az CLI
├── Real-World Patterns
└── Quick reference
```

---

## Part 1: Event Grid Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT GRID OVERVIEW                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Event Grid = Reactive event router                                  │
│ "When something happens → Tell someone about it"                 │
│                                                                       │
│ Example:                                                             │
│ Blob uploaded to Storage → Event Grid → Azure Function runs    │
│ Resource deleted in Azure → Event Grid → Email notification    │
│ IoT device sends data → Event Grid → Process in real-time     │
│                                                                       │
│ Event Grid vs Service Bus vs Event Hub:                            │
│ ┌──────────────┬─────────────┬─────────────┬─────────────┐      │
│ │              │ Event Grid  │ Service Bus │ Event Hub   │      │
│ ├──────────────┼─────────────┼─────────────┼─────────────┤      │
│ │ Purpose      │ React to    │ Reliable msg│ Big data    │      │
│ │              │ events      │ processing  │ streaming   │      │
│ │ Pattern      │ Push (react)│ Pull (queue)│ Pull (stream│      │
│ │ Message size │ 1 MB        │ 256KB-100MB │ 1 MB        │      │
│ │ Throughput   │ 10M events/s│ Moderate    │ Millions/s  │      │
│ │ Ordering     │ No          │ FIFO ✅     │ Per partition│     │
│ │ Retention    │ 24 hours    │ Days-weeks  │ 1-90 days   │      │
│ │ Use case     │ Blob events,│ Order proc, │ Telemetry,  │      │
│ │              │ webhooks    │ workflows   │ logs, IoT   │      │
│ │ Price        │ Per event   │ Per op/month│ Per TU/month│      │
│ └──────────────┴─────────────┴─────────────┴─────────────┘      │
│                                                                       │
│ Pricing: First 100,000 events/month FREE!                         │
│ Then: $0.60 per million events                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating Event Subscriptions (Full Portal Walkthrough)

```
Example: Trigger a Function when a blob is uploaded

Storage Account → Events → [+ Event Subscription]

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE EVENT SUBSCRIPTION                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Basic tab ──                                                     │
│                                                                       │
│ Name: [blob-uploaded-handler]                                       │
│ ⚡ Descriptive name for what this subscription does               │
│                                                                       │
│ Event Schema: [Event Grid Schema ▼]                               │
│ ├── Event Grid Schema (default — Azure's native format)         │
│ ├── Cloud Events v1.0 (open standard — CNCF)                   │
│ └── Custom Input Schema (your own format)                        │
│ ⚡ Use Cloud Events for interoperability with other platforms.   │
│                                                                       │
│ Topic type: Storage account (auto-filled from source)              │
│ Source resource: [stmyappprod ▼]                                  │
│                                                                       │
│ System Topic Name: [stmyappprod-events]                            │
│ ⚡ Created automatically when first subscription is created.     │
│                                                                       │
│ Filter to event types:                                              │
│ ☑ Blob Created                                                    │
│ ☐ Blob Deleted                                                    │
│ ☐ Blob Renamed                                                    │
│ ☐ Blob Tier Changed                                               │
│ ⚡ Only select the events you actually need!                      │
│                                                                       │
│ ── Endpoint tab ──                                                  │
│                                                                       │
│ Endpoint type: [Azure Function ▼]                                 │
│ Options:                                                             │
│ ├── Azure Function → Run serverless code                        │
│ ├── Webhook → Any HTTP endpoint (your API, etc.)               │
│ ├── Event Hub → Stream to big data pipeline                    │
│ ├── Service Bus Queue → Queue for reliable processing          │
│ ├── Service Bus Topic → Fan-out to multiple subscribers        │
│ ├── Storage Queue → Simple, cheap queue processing             │
│ └── Logic App → Run automated workflows                        │
│                                                                       │
│ Endpoint: [func-process-image / ProcessBlob ▼]                   │
│ ⚡ Select the specific function within the Function App.          │
│                                                                       │
│ ── Filters tab ──                                                   │
│                                                                       │
│ Enable subject filtering: ☑                                       │
│ Subject begins with: [/blobServices/default/containers/images]   │
│ Subject ends with: [.jpg]                                          │
│ ⚡ Only process JPG files from the "images" container!            │
│                                                                       │
│ Enable advanced filtering: ☐                                      │
│ ⚡ Can filter on event data properties (see Part 5)              │
│                                                                       │
│ ── Additional Features tab ──                                       │
│                                                                       │
│ Event delivery schema: [Event Grid Schema]                        │
│                                                                       │
│ Retry policy:                                                        │
│ ├── Max delivery attempts: [30]                                  │
│ ├── Event TTL (minutes): [1440] (24 hours)                      │
│ └── ⚡ Events retried with exponential backoff                   │
│                                                                       │
│ Dead-lettering:                                                      │
│ ├── Storage account: [stmyappprod ▼]                            │
│ ├── Container: [dead-letter-events]                              │
│ └── ⚡ Failed events saved here for debugging                    │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ Result: Every time a JPG is uploaded to images container →        │
│ Function automatically runs to process it!                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Event Sources & Handlers

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT SOURCES (Who publishes events?)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Built-in event sources (system topics):                             │
│ ├── Storage Account → Blob created/deleted/renamed/tier changed  │
│ ├── Resource Groups → Resource created/updated/deleted           │
│ ├── Subscriptions → Any management operation                     │
│ ├── Container Registry → Image pushed/deleted/quarantined       │
│ ├── Event Hub → Capture file created                             │
│ ├── IoT Hub → Device created/deleted/connected/telemetry        │
│ ├── Service Bus → Message available (active messages)           │
│ ├── Media Services → Job state changed                           │
│ ├── Azure Maps → Geofence entered/exited                        │
│ ├── Key Vault → Secret/key/cert near expiry, rotated           │
│ ├── App Configuration → Key-value modified/deleted              │
│ ├── Machine Learning → Model registered/deployed                │
│ ├── Azure SignalR → Client connected/disconnected               │
│ ├── Azure Policy → State changed (compliant/non-compliant)     │
│ └── API Management → Subscription created/updated               │
│                                                                       │
│ Custom Topics: Publish YOUR OWN events via HTTP POST             │
│ Domain Topics: Organize multiple custom topics under one domain  │
│ Partner Topics: Third-party events (Auth0, SAP, etc.)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           EVENT HANDLERS (Who reacts to events?)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Event handlers (subscribers):                                        │
│ ├── Azure Functions → Run serverless code (most common!)        │
│ ├── Logic Apps → Run automated workflows (no-code)              │
│ ├── Webhook → Call any HTTP endpoint (your API, external)      │
│ ├── Event Hub → Stream to big data pipeline                    │
│ ├── Service Bus Queue → Queue for reliable processing          │
│ ├── Service Bus Topic → Fan-out to multiple subscribers        │
│ ├── Storage Queue → Simple, cheap queue                        │
│ ├── Power Automate → No-code automation (Microsoft 365)        │
│ └── Partner destinations (Datadog, Confluent, etc.)            │
│                                                                       │
│ How handlers receive events:                                        │
│ ├── PUSH model: Event Grid pushes events to your handler        │
│ ├── No polling needed (unlike queue-based systems)              │
│ ├── Handler must respond within 60 seconds (or event retried)  │
│ └── Handler must return HTTP 200/201/202 to acknowledge         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Custom Topics (Portal Walkthrough)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CREATE CUSTOM TOPIC                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ You can publish YOUR OWN events to Event Grid!                     │
│                                                                       │
│ Console → Event Grid Topics → [+ Create]                          │
│                                                                       │
│ ── Basics tab ──                                                    │
│                                                                       │
│ Subscription: [My Azure Subscription ▼]                           │
│ Resource group: [rg-messaging ▼]                                  │
│ Name: [egt-order-events]                                           │
│ Region: [Central India ▼]                                         │
│ ⚡ Choose the same region as your event sources/handlers.         │
│                                                                       │
│ ── Networking tab ──                                                │
│                                                                       │
│ Connectivity method:                                                 │
│ ● Public access (allow all networks)                              │
│ ○ Private access (private endpoints only)                        │
│ ⚡ For production, consider private endpoints for security.       │
│                                                                       │
│ ── Advanced tab ──                                                  │
│                                                                       │
│ Event Schema:                                                        │
│ ● Event Grid Schema (Azure native — default)                    │
│ ○ Cloud Events v1.0 (CNCF standard — portable)                 │
│ ○ Custom Event Schema (your own format)                         │
│ ⚡ Cloud Events is recommended for multi-platform scenarios.     │
│                                                                       │
│ [Review + create] → [Create]                                      │
│                                                                       │
│ After creation:                                                      │
│ ├── Topic Endpoint (URL to publish events to):                   │
│ │   https://egt-order-events.centralindia-1.eventgrid.azure.net │
│ │   /api/events                                                   │
│ ├── Access Keys (for authentication):                            │
│ │   Key 1: xxxxxxxxxxxxxxxxxx                                    │
│ │   Key 2: yyyyyyyyyyyyyyyyyy                                    │
│ └── ⚡ Use either key in the aeg-sas-key header                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Publishing Custom Events

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUBLISH YOUR OWN EVENTS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ HTTP POST to your topic endpoint:                                   │
│                                                                       │
│ POST https://egt-order-events.../api/events                       │
│ Content-Type: application/json                                      │
│ aeg-sas-key: <access-key>                                          │
│                                                                       │
│ [{                                                                   │
│   "id": "unique-id-123",                                           │
│   "eventType": "Order.Created",                                    │
│   "subject": "/orders/12345",                                      │
│   "data": {                                                         │
│     "orderId": "12345",                                            │
│     "customerId": "C001",                                          │
│     "total": 99.99,                                                │
│     "items": ["Widget A", "Widget B"]                             │
│   },                                                                 │
│   "dataVersion": "1.0",                                            │
│   "eventTime": "2024-01-15T10:30:00Z"                             │
│ }]                                                                   │
│                                                                       │
│ ⚡ You can send up to 1 MB per request (multiple events)          │
│ ⚡ Each individual event max: 1 MB                                │
│ ⚡ Response: 200 OK = events accepted                             │
│                                                                       │
│ Then create subscriptions on this topic to handle these events!   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Filtering & Dead-Letter

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT FILTERING                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Event Subscription → Filters tab                                   │
│                                                                       │
│ Subject filtering (simple):                                         │
│ ├── Subject begins with: /blobServices/default/containers/images│
│ ├── Subject ends with: .jpg                                      │
│ └── Result: Only fires for JPG files in "images" container!     │
│                                                                       │
│ Advanced filtering (powerful):                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Key                  │ Operator         │ Value              │  │
│ ├──────────────────────┼──────────────────┼────────────────────┤  │
│ │ data.orderTotal      │ GreaterThan      │ 100                │  │
│ │ data.category        │ StringIn         │ ["electronics",    │  │
│ │                      │                  │  "clothing"]       │  │
│ │ eventType            │ StringContains   │ "Created"          │  │
│ └──────────────────────┴──────────────────┴────────────────────┘  │
│                                                                       │
│ Advanced filter operators:                                          │
│ ├── NumberGreaterThan, NumberLessThan, NumberIn                  │
│ ├── StringBeginsWith, StringEndsWith, StringContains            │
│ ├── StringIn, StringNotIn                                        │
│ ├── BoolEquals                                                    │
│ ├── IsNullOrUndefined, IsNotNull                                │
│ └── Combine up to 25 advanced filters per subscription         │
│                                                                       │
│ ⚡ Filtering reduces costs — you only pay for delivered events! │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DEAD-LETTERING & RETRY                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Event Subscription → Additional features                           │
│                                                                       │
│ When delivery fails:                                                 │
│ 1. Event Grid retries with exponential backoff                    │
│ 2. Retry schedule: 10s, 30s, 1m, 5m, 10m, 30m, 1h, then hourly│
│ 3. After max retries or TTL → Dead-letter OR drop                │
│                                                                       │
│ Dead-letter configuration:                                          │
│ ├── Storage account: [stmyappprod ▼]                            │
│ ├── Container: [dead-letter-events]                              │
│ ├── Events saved as JSON blobs in the container                 │
│ └── Path: {topic}/{subscription}/{year}/{month}/{day}/{hour}    │
│                                                                       │
│ Retry policy:                                                        │
│ ├── Max delivery attempts: [30] (1 to 30)                       │
│ ├── Event TTL (minutes): [1440] (1 to 1440 = 24 hours)        │
│ └── Event dropped after whichever limit is hit first            │
│                                                                       │
│ ⚡ Without dead-letter: Failed events are silently dropped!      │
│ ⚡ Always enable dead-letter for production workloads.           │
│ ⚡ Monitor the dead-letter container for delivery failures.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Managing & Deleting Event Subscriptions

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING EVENT SUBSCRIPTIONS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ View all subscriptions:                                              │
│ Event Grid → Event Subscriptions → Lists all subscriptions       │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Name                 │ Topic         │ Handler     │ Status │  │
│ ├──────────────────────┼───────────────┼─────────────┼────────┤  │
│ │ blob-uploaded        │ stmyappprod   │ Function    │ Active │  │
│ │ order-handler        │ egt-orders    │ Webhook     │ Active │  │
│ │ resource-monitor     │ rg-prod       │ Logic App   │ Active │  │
│ └──────────────────────┴───────────────┴─────────────┴────────┘  │
│                                                                       │
│ Click subscription → View:                                         │
│ ├── Delivery statistics (succeeded, failed, expired)            │
│ ├── Dead-letter count                                            │
│ ├── Endpoint health                                               │
│ └── Last delivery attempt timestamp                               │
│                                                                       │
│ Manage topics:                                                       │
│ Event Grid → Topics → [topic-name] →                             │
│ ├── Event Subscriptions: Add/view/delete subscriptions          │
│ ├── Access Keys: View/regenerate keys                           │
│ ├── Access Control (IAM): Manage permissions                    │
│ └── Metrics: View delivery stats, published events, etc.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DELETING EVENT SUBSCRIPTIONS & TOPICS                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Delete event subscription (Portal):                                 │
│ Event Grid → Event Subscriptions → [subscription] → [Delete]    │
│ ⚡ Events are no longer delivered. No pending events are lost.   │
│                                                                       │
│ Delete custom topic (Portal):                                       │
│ Event Grid → Topics → [topic] → [Delete]                        │
│ ⚡ All subscriptions under this topic are also deleted!          │
│ ⚡ Events can no longer be published to this topic.              │
│                                                                       │
│ Delete system topic (Portal):                                       │
│ Event Grid → System Topics → [topic] → [Delete]                 │
│ ⚡ Source resource keeps working, just no event routing.         │
│                                                                       │
│ CLI:                                                                 │
│   # Delete event subscription                                     │
│   az eventgrid event-subscription delete \                        │
│     --name blob-handler \                                          │
│     --source-resource-id <resource-id>                            │
│                                                                       │
│   # Delete custom topic (and all its subscriptions)              │
│   az eventgrid topic delete \                                      │
│     --name egt-order-events \                                      │
│     --resource-group rg-messaging                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
# Custom Topic
resource "azurerm_eventgrid_topic" "orders" {
  name                = "egt-order-events"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  tags = {
    environment = "production"
  }
}

# System Topic (for Storage Account events)
resource "azurerm_eventgrid_system_topic" "storage" {
  name                   = "egt-storage-events"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  source_arm_resource_id = azurerm_storage_account.main.id
  topic_type             = "Microsoft.Storage.StorageAccounts"
}

# Event Subscription on System Topic (Storage → Function)
resource "azurerm_eventgrid_system_topic_event_subscription" "blob" {
  name                = "blob-created-handler"
  system_topic        = azurerm_eventgrid_system_topic.storage.name
  resource_group_name = azurerm_resource_group.main.name

  included_event_types = ["Microsoft.Storage.BlobCreated"]

  subject_filter {
    subject_begins_with = "/blobServices/default/containers/images"
    subject_ends_with   = ".jpg"
  }

  azure_function_endpoint {
    function_id = "${azurerm_linux_function_app.main.id}/functions/ProcessBlob"
  }

  dead_letter_identity {
    type = "SystemAssigned"
  }

  storage_blob_dead_letter_destination {
    storage_account_id          = azurerm_storage_account.main.id
    storage_blob_container_name = "dead-letter-events"
  }

  retry_policy {
    max_delivery_attempts = 30
    event_time_to_live    = 1440
  }
}
```

### Bicep

```bicep
// Custom Topic
resource eventGridTopic 'Microsoft.EventGrid/topics@2023-12-15-preview' = {
  name: 'egt-order-events'
  location: resourceGroup().location
  properties: {
    inputSchema: 'EventGridSchema'
    publicNetworkAccess: 'Enabled'
  }
  tags: {
    environment: 'production'
  }
}

// System Topic (for Storage Account events)
resource systemTopic 'Microsoft.EventGrid/systemTopics@2023-12-15-preview' = {
  name: 'egt-storage-events'
  location: resourceGroup().location
  properties: {
    source: storageAccount.id
    topicType: 'Microsoft.Storage.StorageAccounts'
  }
}

// Event Subscription on System Topic
resource eventSubscription 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2023-12-15-preview' = {
  parent: systemTopic
  name: 'blob-created-handler'
  properties: {
    destination: {
      endpointType: 'AzureFunction'
      properties: {
        resourceId: '${functionApp.id}/functions/ProcessBlob'
      }
    }
    filter: {
      includedEventTypes: [
        'Microsoft.Storage.BlobCreated'
      ]
      subjectBeginsWith: '/blobServices/default/containers/images'
      subjectEndsWith: '.jpg'
    }
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440
    }
    deadLetterDestination: {
      endpointType: 'StorageBlob'
      properties: {
        resourceId: storageAccount.id
        blobContainerName: 'dead-letter-events'
      }
    }
  }
}

// Output topic endpoint and key
output topicEndpoint string = eventGridTopic.properties.endpoint
output topicKey string = listKeys(eventGridTopic.id, '2023-12-15-preview').key1
```

---

## Part 8: az CLI Reference

```bash
# ─── CUSTOM TOPICS ───

# Create custom topic
az eventgrid topic create \
  --name egt-order-events \
  --resource-group rg-messaging \
  --location centralindia

# List topics
az eventgrid topic list --resource-group rg-messaging --output table

# Show topic details (get endpoint URL)
az eventgrid topic show \
  --name egt-order-events \
  --resource-group rg-messaging

# Get topic access keys
az eventgrid topic key list \
  --name egt-order-events \
  --resource-group rg-messaging

# Regenerate topic key
az eventgrid topic key regenerate \
  --name egt-order-events \
  --resource-group rg-messaging \
  --key-name key1

# Update topic (add tags)
az eventgrid topic update \
  --name egt-order-events \
  --resource-group rg-messaging \
  --tags environment=production

# ─── EVENT SUBSCRIPTIONS ───

# Create subscription on custom topic (webhook)
az eventgrid event-subscription create \
  --name order-webhook \
  --source-resource-id /subscriptions/<sub-id>/resourceGroups/rg-messaging/providers/Microsoft.EventGrid/topics/egt-order-events \
  --endpoint https://myapp.azurewebsites.net/api/events \
  --endpoint-type webhook \
  --included-event-types Order.Created Order.Updated

# Create subscription on Storage Account (Function)
az eventgrid event-subscription create \
  --name blob-handler \
  --source-resource-id /subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.Storage/storageAccounts/stmyappprod \
  --endpoint /subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.Web/sites/func-app/functions/ProcessBlob \
  --endpoint-type azurefunction \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/images \
  --subject-ends-with .jpg

# Create subscription with dead-letter
az eventgrid event-subscription create \
  --name order-handler \
  --source-resource-id <topic-resource-id> \
  --endpoint <function-endpoint> \
  --deadletter-endpoint /subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.Storage/storageAccounts/stmyappprod/blobServices/default/containers/dead-letter

# List event subscriptions for a topic
az eventgrid event-subscription list \
  --source-resource-id <topic-resource-id> \
  --output table

# Show subscription details
az eventgrid event-subscription show \
  --name blob-handler \
  --source-resource-id <resource-id>

# Show subscription delivery status
az eventgrid event-subscription show \
  --name blob-handler \
  --source-resource-id <resource-id> \
  --include-full-endpoint-url

# Update subscription (change endpoint)
az eventgrid event-subscription update \
  --name order-handler \
  --source-resource-id <topic-resource-id> \
  --endpoint https://newapp.azurewebsites.net/api/events

# ─── DELETE ───

# Delete event subscription
az eventgrid event-subscription delete \
  --name blob-handler \
  --source-resource-id <resource-id>

# Delete custom topic (and all subscriptions)
az eventgrid topic delete \
  --name egt-order-events \
  --resource-group rg-messaging

# ─── SYSTEM TOPICS ───

# List system topics
az eventgrid system-topic list --resource-group rg-prod --output table

# Create system topic explicitly
az eventgrid system-topic create \
  --name egt-storage-topic \
  --resource-group rg-prod \
  --location centralindia \
  --source /subscriptions/<sub-id>/resourceGroups/rg-prod/providers/Microsoft.Storage/storageAccounts/stmyappprod \
  --topic-type Microsoft.Storage.StorageAccounts

# Delete system topic
az eventgrid system-topic delete \
  --name egt-storage-topic \
  --resource-group rg-prod
```

---

## Part 9: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: AUTO-PROCESS UPLOADED FILES                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ User uploads invoice PDF → Blob Storage                            │
│ → Event Grid (BlobCreated) → Azure Function                      │
│ → Function calls AI (Form Recognizer) to extract data            │
│ → Saves structured data to database                               │
│ → Sends confirmation email via Logic App                          │
│                                                                       │
│ Zero manual processing. Fully automated!                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: RESOURCE COMPLIANCE MONITORING                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Any resource created/modified in subscription                      │
│ → Event Grid (ResourceWriteSuccess) → Azure Function             │
│ → Function checks: Does resource have required tags?              │
│   ├── Yes → Log as compliant ✅                                  │
│   └── No → Send alert to Slack/Teams + apply tags automatically │
│                                                                       │
│ Real-time compliance enforcement (not just periodic scanning).    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: MICROSERVICES EVENT-DRIVEN ARCHITECTURE             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│ │ Order Service│──→ │ Event Grid       │──→ │ Inventory Service│  │
│ │ (publishes   │    │ Topic:           │──→ │ Email Service    │  │
│ │  Order.      │    │ egt-order-events │──→ │ Analytics Service│  │
│ │  Created)    │    │                  │──→ │ Billing Service  │  │
│ └─────────────┘    └──────────────────┘    └──────────────────┘  │
│                                                                       │
│ One event → Multiple handlers (fan-out pattern)                   │
│ Services are fully decoupled (don't know about each other).      │
│ Add new subscribers anytime without changing the publisher!       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: KEY VAULT SECRET ROTATION                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Key Vault secret near expiry (30 days before)                      │
│ → Event Grid (SecretNearExpiry) → Azure Function                 │
│ → Function generates new secret/password                          │
│ → Updates Key Vault with new version                              │
│ → Updates application configuration                               │
│ → Sends notification to ops team                                   │
│                                                                       │
│ Automated secret rotation without downtime!                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT GRID QUICK REFERENCE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Reactive event router ("when X happens, do Y")               │
│ Price: First 100K events/month FREE, then $0.60/million            │
│                                                                       │
│ Topic types:                                                         │
│ ├── System Topics: Built-in Azure events (blob, resource, etc.) │
│ ├── Custom Topics: Your own events via HTTP POST                 │
│ └── Domain Topics: Organize many custom topics under one domain │
│                                                                       │
│ Event Sources: Storage, Resource Groups, IoT Hub, Key Vault, etc. │
│ Event Handlers: Functions, Logic Apps, Webhooks, Event Hub, etc.  │
│                                                                       │
│ Filtering: Subject (begins/ends with) + Advanced (data properties)│
│ Dead-letter: Failed events go to Storage blob for debugging       │
│ Retry: Up to 30 retries, 24-hour TTL, exponential backoff        │
│                                                                       │
│ vs Service Bus: Event Grid = reactive push; Service Bus = queue  │
│ vs Event Hub: Event Grid = discrete events; Event Hub = streams  │
│                                                                       │
│ CLI quick:                                                           │
│ ├── Topic: az eventgrid topic create --name X --rg Y             │
│ ├── Sub:   az eventgrid event-subscription create --name X ...   │
│ ├── Keys:  az eventgrid topic key list --name X --rg Y           │
│ └── Delete: az eventgrid topic delete --name X --rg Y            │
│                                                                       │
│ ⚡ AWS equivalent: Amazon EventBridge                               │
│ ⚡ GCP equivalent: Eventarc                                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Next chapter: [Chapter 50: Azure Event Hub](50-event-hub.md) — Big data streaming with event ingestion at millions of events per second. Learn how Event Hub differs from Event Grid (streaming data vs discrete events) and when to use each.
