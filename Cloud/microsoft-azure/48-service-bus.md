# Chapter 48: Azure Service Bus

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Service Bus Fundamentals](#part-1-service-bus-fundamentals)
- [Part 2: Creating a Service Bus (Portal Walkthrough)](#part-2-creating-a-service-bus-portal-walkthrough)
- [Part 3: Queues](#part-3-queues)
- [Part 4: Topics & Subscriptions](#part-4-topics--subscriptions)
- [Part 5: Advanced Features](#part-5-advanced-features)
- [Part 6: Code Examples](#part-6-code-examples)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Service Bus is an enterprise message broker for reliable, ordered messaging between applications. It decouples services so they don't need to be online at the same time. Service Bus supports queues (point-to-point) and topics (publish-subscribe) patterns.

```
What you'll learn:
├── Service Bus Fundamentals
│   ├── What is a message broker (decouple services)
│   ├── Service Bus vs Storage Queues vs Event Grid vs Event Hub
│   └── Tiers (Basic, Standard, Premium)
├── Creating a Service Bus (Portal)
├── Queues (one sender → one receiver)
├── Topics & Subscriptions (one sender → many receivers)
├── Advanced Features (dead-letter, sessions, dedup, scheduled)
├── Code Examples (Node.js, .NET)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Service Bus Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVICE BUS OVERVIEW                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Why use a message broker?                                           │
│                                                                       │
│ Without Service Bus (tight coupling):                               │
│ Order Service →→→ Payment Service (if Payment is down, order fails!)│
│                                                                       │
│ With Service Bus (decoupled):                                       │
│ Order Service → Queue → Payment Service                           │
│ ⚡ If Payment is down, messages wait in the queue!               │
│ ⚡ Payment processes them when it comes back online.             │
│                                                                       │
│ Messaging comparison:                                                │
│ ┌──────────────────┬────────────┬────────────┬────────────┐      │
│ │ Feature          │ Service Bus│ Storage Q  │ Event Grid │      │
│ ├──────────────────┼────────────┼────────────┼────────────┤      │
│ │ Type             │ Message Q  │ Simple Q   │ Event router│     │
│ │ Ordering         │ FIFO ✅    │ No         │ No          │     │
│ │ Message size     │ 256KB-100MB│ 64KB       │ 1MB         │     │
│ │ Dead-letter      │ Yes ✅     │ No         │ Yes         │     │
│ │ Sessions         │ Yes ✅     │ No         │ No          │     │
│ │ Transactions     │ Yes ✅     │ No         │ No          │     │
│ │ Duplicate detect │ Yes ✅     │ No         │ No          │     │
│ │ Topics/Subs      │ Yes ✅     │ No         │ Yes         │     │
│ │ Price            │ Medium     │ Cheapest   │ Per event   │     │
│ │ Best for         │ Enterprise │ Simple jobs│ Reactions   │     │
│ └──────────────────┴────────────┴────────────┴────────────┘      │
│                                                                       │
│ Tiers:                                                               │
│ ├── Basic: Queues only, no topics (~$0.05/million ops)          │
│ ├── Standard: Queues + Topics (~$10/month + per-op)            │
│ └── Premium: Dedicated, VNet, large messages ($668+/month)     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Service Bus (Portal Walkthrough)

```
Console → Service Bus → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE SERVICE BUS NAMESPACE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-messaging ▼]                                   │
│                                                                       │
│ Namespace name: [sb-myapp-prod]                                    │
│ ⚡ Creates: sb-myapp-prod.servicebus.windows.net                 │
│                                                                       │
│ Location: [Central India ▼]                                        │
│ Pricing tier: [Standard ▼]                                        │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation:                                                      │
│ Get connection string:                                               │
│ Namespace → Shared access policies → RootManageSharedAccessKey  │
│ Copy: Primary Connection String                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Queues

```
┌─────────────────────────────────────────────────────────────────────┐
│           QUEUES (Point-to-Point)                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Namespace → Queues → [+ Queue]                                   │
│                                                                       │
│ Name: [orders-queue]                                                │
│ Max queue size: [1 GB ▼] (1-80 GB)                               │
│ Message TTL: [14 days] (how long messages live)                   │
│ Lock duration: [30 seconds] (time to process before reappearing) │
│ Max delivery count: [10] (retries before dead-letter)            │
│                                                                       │
│ ☑ Enable dead-lettering on message expiration                    │
│ ☐ Enable duplicate detection                                     │
│ ☐ Enable sessions (ordered processing per session)              │
│ ☐ Enable partitioning                                            │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ How queues work:                                                    │
│ Sender → [Message A] [Message B] [Message C] → Receiver       │
│                                                                       │
│ 1. Sender sends message to queue                                  │
│ 2. Message waits in queue                                          │
│ 3. ONE receiver picks up the message (locks it)                  │
│ 4. Receiver processes message → completes it (removed from queue)│
│ 5. If receiver crashes → message reappears after lock timeout  │
│                                                                       │
│ Receive modes:                                                       │
│ ├── Peek-lock (default): Message locked, complete when done    │
│ │   ⚡ Safe! If processing fails, message comes back.          │
│ └── Receive-and-delete: Message removed immediately             │
│     ⚡ Faster but no retry if processing fails.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Topics & Subscriptions

```
┌─────────────────────────────────────────────────────────────────────┐
│           TOPICS & SUBSCRIPTIONS (Publish-Subscribe)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Topic = Broadcast channel (one sender, many receivers)            │
│ Subscription = Each receiver gets its own copy of messages       │
│                                                                       │
│ Namespace → Topics → [+ Topic]                                   │
│ Name: [order-events]                                                │
│                                                                       │
│ Then create subscriptions:                                          │
│ Topic → Subscriptions → [+ Subscription]                        │
│ ├── email-subscription (sends order confirmation emails)        │
│ ├── inventory-subscription (updates stock levels)               │
│ └── analytics-subscription (records order analytics)            │
│                                                                       │
│ How it works:                                                        │
│                         ┌→ email-subscription → Email Service   │
│ Order Service → Topic ──┼→ inventory-subscription → Inventory   │
│                         └→ analytics-subscription → Analytics   │
│                                                                       │
│ ⚡ Each subscription gets a COPY of every message!              │
│ ⚡ Unlike queues (one receiver), topics support many receivers. │
│                                                                       │
│ Subscription filters:                                               │
│ ├── SQL filter: "OrderTotal > 100 AND Region = 'Asia'"         │
│ ├── Correlation filter: Match on message properties            │
│ └── Boolean filter: True (all messages) / False (no messages)  │
│                                                                       │
│ Example: inventory-subscription with SQL filter:                   │
│ "ItemType = 'Physical'" → Only physical orders, not digital    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Advanced Features

```
Dead-Letter Queue (DLQ):
├── Messages that can't be processed go here (after max retries)
├── Every queue/subscription has its own DLQ
├── Inspect DLQ to find and fix processing errors
└── Messages stay in DLQ until manually processed or deleted

Sessions (ordered processing):
├── Group messages by session ID
├── Messages within a session are processed in FIFO order
├── Use case: Process all messages for Order #12345 in sequence
└── Only one receiver can lock a session at a time

Duplicate Detection:
├── Detect and discard duplicate messages (by MessageId)
├── Window: configurable (10 seconds to 7 days)
└── Use case: Sender retries but message was already received

Scheduled Messages:
├── Send message now, but it appears in queue at scheduled time
├── Use case: "Send reminder email in 24 hours"
└── message.ScheduledEnqueueTimeUtc = DateTime.UtcNow.AddHours(24)

Auto-Forwarding:
├── Automatically forward messages from one queue/subscription to another
├── Use case: Fan-out, chaining, centralized logging
└── Topic → Subscription → Auto-forward to another queue

Message Batching:
├── Send multiple messages in one network call
├── Reduces network overhead
└── sender.SendMessagesAsync(messageBatch)
```

---

## Part 6: Code Examples

### Node.js

```javascript
const { ServiceBusClient } = require("@azure/service-bus");

const connectionString = process.env.SERVICE_BUS_CONNECTION_STRING;
const client = new ServiceBusClient(connectionString);

// Send message
async function sendMessage() {
  const sender = client.createSender("orders-queue");
  await sender.sendMessages({
    body: { orderId: "12345", amount: 99.99 },
    contentType: "application/json",
  });
  await sender.close();
}

// Receive messages
async function receiveMessages() {
  const receiver = client.createReceiver("orders-queue");
  const messages = await receiver.receiveMessages(10, { maxWaitTimeInMs: 5000 });
  for (const msg of messages) {
    console.log("Order:", msg.body);
    await receiver.completeMessage(msg); // Remove from queue
  }
  await receiver.close();
}
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_servicebus_namespace" "main" {
  name                = "sb-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
}

resource "azurerm_servicebus_queue" "orders" {
  name         = "orders-queue"
  namespace_id = azurerm_servicebus_namespace.main.id
  max_delivery_count = 10
  dead_lettering_on_message_expiration = true
}

resource "azurerm_servicebus_topic" "events" {
  name         = "order-events"
  namespace_id = azurerm_servicebus_namespace.main.id
}

resource "azurerm_servicebus_subscription" "email" {
  name     = "email-subscription"
  topic_id = azurerm_servicebus_topic.events.id
  max_delivery_count = 10
}
```

### Bicep

```bicep
// Service Bus Namespace
resource serviceBus 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-myapp-prod'
  location: resourceGroup().location
  sku: { name: 'Standard', tier: 'Standard' }
}

// Queue
resource queue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: serviceBus
  name: 'order-processing'
  properties: {
    maxDeliveryCount: 10
    lockDuration: 'PT1M'
    deadLetteringOnMessageExpiration: true
    maxSizeInMegabytes: 1024
  }
}

// Topic
resource topic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: serviceBus
  name: 'order-events'
  properties: {
    maxSizeInMegabytes: 1024
    defaultMessageTimeToLive: 'P14D'
  }
}

// Subscription
resource subscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: topic
  name: 'email-subscription'
  properties: {
    maxDeliveryCount: 10
    lockDuration: 'PT1M'
    deadLetteringOnMessageExpiration: true
  }
}
```
  --resource-group rg-messaging \
  --sku Standard

# Create queue
az servicebus queue create \
  --namespace-name sb-myapp-prod \
  --resource-group rg-messaging \
  --name orders-queue

# Create topic
az servicebus topic create \
  --namespace-name sb-myapp-prod \
  --resource-group rg-messaging \
  --name order-events

# Create subscription
az servicebus topic subscription create \
  --namespace-name sb-myapp-prod \
  --resource-group rg-messaging \
  --topic-name order-events \
  --name email-subscription

# Get connection string
az servicebus namespace authorization-rule keys list \
  --namespace-name sb-myapp-prod \
  --resource-group rg-messaging \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv

# Delete namespace
az servicebus namespace delete --name sb-myapp-prod --resource-group rg-messaging
```

---

## Real-World Patterns

### Pattern 1: Order Processing with Sessions

```
┌─────────────────────────────────────────────────┐
│       Order Processing Pipeline                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  Web API ──→ Service Bus Queue (Sessions)       │
│              (Session ID = Order ID)            │
│                    │                            │
│         ┌──────────┼──────────┐                 │
│         ▼          ▼          ▼                 │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│   │Worker 1  │ │Worker 2  │ │Worker 3  │       │
│   │Order-101 │ │Order-102 │ │Order-103 │       │
│   └──────────┘ └──────────┘ └──────────┘       │
│                                                 │
│  Why Sessions?                                  │
│  - All messages for same order go to same       │
│    worker (FIFO per order)                      │
│  - Different orders processed in parallel       │
│  - No out-of-order issues                       │
└─────────────────────────────────────────────────┘
```

### Pattern 2: Pub/Sub with Topic Filters

```
┌─────────────────────────────────────────────────┐
│       Event Distribution with Topics            │
├─────────────────────────────────────────────────┤
│                                                 │
│  Payment Service ──→ Topic: "payments"           │
│                         │                       │
│              ┌──────────┼──────────┐            │
│              ▼          ▼          ▼            │
│        ┌──────────┐┌──────────┐┌──────────┐    │
│        │Sub: Email ││Sub: Audit││Sub: Fraud│    │
│        │Filter:   ││Filter:   ││Filter:   │    │
│        │amount>0  ││all       ││amount    │    │
│        │          ││          ││  >10000  │    │
│        └──────────┘└──────────┘└──────────┘    │
│                                                 │
│  Each subscription gets its own copy of         │
│  matching messages. Dead-letter queue catches   │
│  processing failures automatically.             │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Service Bus = Enterprise message broker (reliable, ordered)

Queues: Point-to-point (one sender → one receiver)
Topics: Publish-subscribe (one sender → many receivers via subscriptions)

Tiers: Basic (queues only) | Standard (+ topics) | Premium (dedicated)

Key features:
  Dead-letter queue (failed messages)
  Sessions (ordered processing per group)
  Duplicate detection
  Scheduled messages
  Peek-lock (safe processing with retry)

Receive modes: Peek-lock (safe) | Receive-and-delete (fast)
Subscription filters: SQL filter | Correlation filter
```

---

## What's Next?

Next chapter: [Chapter 49: Azure Event Grid](49-event-grid.md) — Event-driven architecture with publish-subscribe event routing.
