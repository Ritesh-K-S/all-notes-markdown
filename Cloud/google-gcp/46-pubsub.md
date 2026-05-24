# Chapter 46 — Pub/Sub

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Pub/Sub Fundamentals](#part-1--pubsub-fundamentals)
- [Part 2: Topics](#part-2--topics)
- [Part 3: Subscriptions — Pull Mode](#part-3--subscriptions--pull-mode)
- [Part 4: Subscriptions — Push Mode](#part-4--subscriptions--push-mode)
- [Part 5: Message Ordering](#part-5--message-ordering)
- [Part 6: Message Filtering](#part-6--message-filtering)
- [Part 7: Dead-Letter Topics](#part-7--dead-letter-topics)
- [Part 8: Schemas & Message Validation](#part-8--schemas--message-validation)
- [Part 9: Exactly-Once Delivery](#part-9--exactly-once-delivery)
- [Part 10: BigQuery Subscriptions](#part-10--bigquery-subscriptions)
- [Part 11: Cloud Storage Subscriptions](#part-11--cloud-storage-subscriptions)
- [Part 12: Retention, Replay & Snapshots](#part-12--retention-replay--snapshots)
- [Part 13: IAM & Security](#part-13--iam--security)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What is Pub/Sub? (Beginner Explanation)](#what-is-pubsub-beginner-explanation)
- [Console Walkthrough: Creating & Managing Topics](#console-walkthrough-creating--managing-topics)
- [Monitoring & Troubleshooting](#monitoring--troubleshooting)
- [What's Next?](#whats-next)

---

## Overview

Cloud Pub/Sub is Google Cloud's fully managed, real-time messaging service that allows you to send and receive messages between independent applications. It follows the publish-subscribe pattern: publishers send messages to topics, and subscribers receive messages from subscriptions attached to those topics. Pub/Sub guarantees at-least-once delivery, supports ordering, filtering, dead-letter queues, schema validation, and scales automatically to handle millions of messages per second.

---

## Part 1 — Pub/Sub Fundamentals

### What Is Pub/Sub?

```
┌─────────────────────────────────────────────────────────────────┐
│              PUB/SUB OVERVIEW                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Publisher-Subscriber pattern:                                   │
│                                                                   │
│  ┌──────────┐    ┌───────────┐    ┌──────────────┐             │
│  │Publisher A│───►│           │───►│Subscription 1│──►Subscriber│
│  └──────────┘    │   Topic   │    └──────────────┘             │
│  ┌──────────┐───►│           │    ┌──────────────┐             │
│  │Publisher B│    │           │───►│Subscription 2│──►Subscriber│
│  └──────────┘    └───────────┘    └──────────────┘             │
│                                   ┌──────────────┐             │
│                              ───►│Subscription 3│──►Subscriber│
│                                   └──────────────┘             │
│                                                                   │
│  Key concepts:                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Topic        → Named channel for messages                │   │
│  │ Subscription → Named attachment to a topic               │   │
│  │ Publisher    → Sends messages to a topic                  │   │
│  │ Subscriber   → Receives messages from a subscription     │   │
│  │ Message      → Data (body) + attributes (key-value)      │   │
│  │ Ack          → Subscriber confirms message processed     │   │
│  │ Ack Deadline → Time to ack before redelivery (10-600s)   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Each subscription gets its OWN copy of every message.          │
│  Within a subscription, each message delivered to ONE consumer. │
│  This enables both fan-out and load-balanced consumption.       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Pub/Sub | AWS SNS + SQS | Azure Service Bus |
|---------|------------|----------------|-------------------|
| Service | Pub/Sub (unified) | SNS (topics) + SQS (queues) | Service Bus |
| Publish | Topic | SNS Topic | Topic |
| Subscribe | Subscription | SQS Queue | Subscription |
| Pull | Yes | SQS ReceiveMessage | Receive |
| Push | Yes (HTTP endpoint) | SNS → HTTP/Lambda | N/A (use Functions) |
| Ordering | Yes (ordering key) | SQS FIFO | Yes (sessions) |
| Filtering | Yes (attributes) | SNS filter policies | Yes (SQL filters) |
| Dead-letter | Yes | Yes (SQS DLQ) | Yes |
| Schema | Yes (Avro, Protobuf) | No | No |
| BigQuery sink | Yes (native) | No (need Firehose) | No |
| Exactly-once | Yes | SQS FIFO only | Yes (sessions) |
| Throughput | Unlimited (auto-scale) | Varies by service | Varies by tier |

### Pricing

| Component | Cost |
|-----------|------|
| Message ingestion | $40/TiB |
| Message delivery | $40/TiB (pull), varies (push) |
| Seek-related storage | $0.27/GiB/month |
| Topic message storage (retention) | $0.27/GiB/month |
| Snapshot message backlog | $0.27/GiB/month |
| Schema registry | Free (first 100 schemas) |
| Free tier | First 10 GiB/month |

---

## Part 2 — Topics

### Creating Topics

```bash
# Create a topic
gcloud pubsub topics create orders-topic

# Create topic with message retention
gcloud pubsub topics create events-topic \
    --message-retention-duration=7d

# Create topic with schema
gcloud pubsub topics create validated-topic \
    --schema=order-schema \
    --message-encoding=json

# Create topic with CMEK encryption
gcloud pubsub topics create secure-topic \
    --topic-encryption-key=projects/my-project/locations/us-central1/keyRings/ring/cryptoKeys/key

# List topics
gcloud pubsub topics list

# Describe topic
gcloud pubsub topics describe orders-topic

# Delete topic
gcloud pubsub topics delete orders-topic

# Publish a message
gcloud pubsub topics publish orders-topic \
    --message='{"orderId": "12345", "amount": 99.99}' \
    --attribute=type=order,priority=high
```

### Topic Architecture

```
┌──────────────────────────────────────────────────────────────┐
│         TOPIC DETAILS                                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Topic: projects/my-project/topics/orders-topic               │
│                                                                │
│  Properties:                                                   │
│  ├── Message retention: 0 (default) to 31 days               │
│  │   → 0 = messages kept only until acked by ALL subs        │
│  │   → >0 = messages kept even after acking (for replay)     │
│  ├── Schema: optional (Avro, Protocol Buffers)                │
│  ├── Encryption: Google-managed or CMEK                       │
│  ├── Labels: key-value pairs for organization                │
│  └── Subscriptions: can have many subscriptions attached     │
│                                                                │
│  Message structure:                                            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  {                                                    │    │
│  │    "data": "base64-encoded-payload",                 │    │
│  │    "attributes": {                                    │    │
│  │      "type": "order",                                │    │
│  │      "priority": "high"                               │    │
│  │    },                                                 │    │
│  │    "messageId": "1234567890",                        │    │
│  │    "publishTime": "2024-01-15T10:30:00Z",            │    │
│  │    "orderingKey": "customer-123"  // optional        │    │
│  │  }                                                    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Max message size: 10 MiB                                     │
│  Max attributes: 100 per message                              │
│  Max attribute key size: 256 bytes                            │
│  Max attribute value size: 1024 bytes                         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 3 — Subscriptions — Pull Mode

### How Pull Works

```
┌──────────────────────────────────────────────────────────────┐
│         PULL SUBSCRIPTION                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Subscriber actively pulls messages from the subscription:   │
│                                                                │
│  ┌────────────┐    pull request     ┌───────────────┐       │
│  │ Subscriber │ ──────────────────► │ Subscription  │       │
│  │            │ ◄────────────────── │               │       │
│  │            │    messages          │ (stores unacked│       │
│  │            │ ──────────────────► │  messages)    │       │
│  │            │    acknowledge       │               │       │
│  └────────────┘                     └───────────────┘       │
│                                                                │
│  Two pull modes:                                               │
│  1. Unary pull (simple, one-shot):                           │
│     → gcloud pubsub subscriptions pull SUB --auto-ack       │
│                                                                │
│  2. Streaming pull (persistent connection):                   │
│     → Client maintains open bidirectional stream              │
│     → Lower latency, higher throughput                       │
│     → Recommended for production                              │
│                                                                │
│  Flow control:                                                 │
│  → Subscriber controls how many messages to receive          │
│  → Set max outstanding messages / bytes                      │
│  → Prevents subscriber from being overwhelmed               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create pull subscription
gcloud pubsub subscriptions create orders-sub \
    --topic=orders-topic \
    --ack-deadline=60

# Pull messages
gcloud pubsub subscriptions pull orders-sub --auto-ack --limit=10

# Pull without auto-ack (manual ack)
gcloud pubsub subscriptions pull orders-sub --limit=5
# Then ack by ID
gcloud pubsub subscriptions ack orders-sub --ack-ids=ACK_ID

# Configure retry policy
gcloud pubsub subscriptions create orders-sub \
    --topic=orders-topic \
    --ack-deadline=60 \
    --min-retry-delay=10s \
    --max-retry-delay=600s

# Configure message retention on subscription
gcloud pubsub subscriptions create orders-sub \
    --topic=orders-topic \
    --retain-acked-messages \
    --message-retention-duration=7d
```

### Pull Subscriber (Python)

```python
from google.cloud import pubsub_v1
from concurrent.futures import TimeoutError

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "orders-sub")

def callback(message):
    print(f"Received: {message.data.decode('utf-8')}")
    print(f"Attributes: {message.attributes}")
    message.ack()  # Acknowledge the message

# Streaming pull with flow control
flow_control = pubsub_v1.types.FlowControl(
    max_messages=100,           # Max outstanding messages
    max_bytes=10 * 1024 * 1024, # Max outstanding bytes (10 MiB)
)

streaming_pull = subscriber.subscribe(
    subscription_path,
    callback=callback,
    flow_control=flow_control,
)

print(f"Listening on {subscription_path}...")
try:
    streaming_pull.result(timeout=300)  # Block for 5 min
except TimeoutError:
    streaming_pull.cancel()
    streaming_pull.result()
```

---

## Part 4 — Subscriptions — Push Mode

### How Push Works

```
┌──────────────────────────────────────────────────────────────┐
│         PUSH SUBSCRIPTION                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Pub/Sub pushes messages to a subscriber's HTTP endpoint:    │
│                                                                │
│  ┌───────────────┐  HTTP POST    ┌────────────────────┐     │
│  │ Subscription  │ ────────────► │ Subscriber         │     │
│  │               │               │ (HTTPS endpoint)   │     │
│  │               │ ◄──────────── │                    │     │
│  │               │  200 OK = ack │ Cloud Run          │     │
│  │               │  4xx/5xx =    │ Cloud Functions    │     │
│  │               │  nack/retry   │ App Engine         │     │
│  └───────────────┘               │ Any HTTPS server   │     │
│                                  └────────────────────┘     │
│                                                                │
│  Push delivers messages as HTTP POST requests:               │
│  → Body contains the Pub/Sub message (JSON)                 │
│  → Subscriber returns 2xx to acknowledge                     │
│  → Any other status → message redelivered                   │
│                                                                │
│  When to use push:                                             │
│  → Serverless subscribers (Cloud Run, Functions)              │
│  → Webhook-style integrations                                 │
│  → Subscriber behind a firewall (no outbound pull needed)    │
│  → Multiple subscribers don't need to manage connections     │
│                                                                │
│  When to use pull:                                             │
│  → High throughput / batch processing                        │
│  → Subscriber needs flow control                             │
│  → Multiple worker instances sharing workload                │
│  → Complex processing with variable ack times                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create push subscription
gcloud pubsub subscriptions create orders-push-sub \
    --topic=orders-topic \
    --push-endpoint="https://my-service-abc123.run.app/pubsub" \
    --ack-deadline=60

# Push with authentication (recommended for Cloud Run/Functions)
gcloud pubsub subscriptions create orders-push-sub \
    --topic=orders-topic \
    --push-endpoint="https://my-service-abc123.run.app/pubsub" \
    --push-auth-service-account=pubsub-invoker@my-project.iam.gserviceaccount.com \
    --push-auth-token-audience="https://my-service-abc123.run.app"

# Update push endpoint
gcloud pubsub subscriptions update orders-push-sub \
    --push-endpoint="https://new-service.run.app/pubsub"
```

### Push Message Format

```json
// HTTP POST body sent by Pub/Sub to push endpoint
{
  "message": {
    "data": "eyJvcmRlcklkIjogIjEyMzQ1In0=",  // base64
    "attributes": {
      "type": "order",
      "priority": "high"
    },
    "messageId": "1234567890",
    "publishTime": "2024-01-15T10:30:00.000Z",
    "orderingKey": ""
  },
  "subscription": "projects/my-project/subscriptions/orders-push-sub"
}
```

---

## Part 5 — Message Ordering

### Ordered Delivery

```
┌──────────────────────────────────────────────────────────────┐
│         MESSAGE ORDERING                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  By default, Pub/Sub does NOT guarantee message order.       │
│  Enable ordering to get messages in publish order:           │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Publisher                                            │    │
│  │  ├── Message 1 (orderingKey="customer-A") ──┐        │    │
│  │  ├── Message 2 (orderingKey="customer-A") ──┤        │    │
│  │  ├── Message 3 (orderingKey="customer-B") ──┼──►Topic│    │
│  │  ├── Message 4 (orderingKey="customer-A") ──┤        │    │
│  │  └── Message 5 (orderingKey="customer-B") ──┘        │    │
│  │                                                      │    │
│  │  Subscription (ordering enabled):                    │    │
│  │  → customer-A messages: 1, 2, 4 (in order)         │    │
│  │  → customer-B messages: 3, 5 (in order)             │    │
│  │  → Messages ACROSS keys may interleave              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Rules:                                                        │
│  • Ordering is per ordering key, not global                  │
│  • Subscription must enable message ordering                 │
│  • Publisher must set ordering key on messages                │
│  • If a message fails to ack, subsequent messages with       │
│    the same key are BLOCKED until the failed one is acked    │
│  • Reduces throughput (ordering adds constraints)            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create subscription with ordering enabled
gcloud pubsub subscriptions create ordered-sub \
    --topic=orders-topic \
    --enable-message-ordering

# Publish with ordering key
gcloud pubsub topics publish orders-topic \
    --message='{"action": "create"}' \
    --ordering-key="customer-123"

gcloud pubsub topics publish orders-topic \
    --message='{"action": "update"}' \
    --ordering-key="customer-123"
```

---

## Part 6 — Message Filtering

### Subscription Filters

```
┌──────────────────────────────────────────────────────────────┐
│         MESSAGE FILTERING                                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Filter messages at the subscription level using attributes: │
│                                                                │
│  Topic receives ALL messages:                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Message 1: {attributes: {type: "order"}}            │    │
│  │  Message 2: {attributes: {type: "notification"}}     │    │
│  │  Message 3: {attributes: {type: "order"}}            │    │
│  │  Message 4: {attributes: {type: "alert"}}            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Subscription A (filter: attributes.type = "order"):         │
│  → Receives: Message 1, Message 3                             │
│                                                                │
│  Subscription B (filter: attributes.type = "notification"):  │
│  → Receives: Message 2                                        │
│                                                                │
│  Subscription C (no filter):                                  │
│  → Receives: ALL messages                                     │
│                                                                │
│  ⚠ Filtered-out messages are auto-acked (not redelivered)    │
│  ⚠ You still pay for filtered-out messages                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create filtered subscription
gcloud pubsub subscriptions create order-sub \
    --topic=events-topic \
    --filter='attributes.type = "order"'

# Filter with multiple conditions
gcloud pubsub subscriptions create high-priority-sub \
    --topic=events-topic \
    --filter='attributes.type = "order" AND attributes.priority = "high"'

# Filter with OR / NOT / HAS
gcloud pubsub subscriptions create alerts-sub \
    --topic=events-topic \
    --filter='attributes.type = "alert" OR attributes.type = "notification"'

# Filter by hasAttribute
gcloud pubsub subscriptions create tagged-sub \
    --topic=events-topic \
    --filter='hasPrefix(attributes.env, "prod")'
```

### Filter Syntax

| Expression | Example |
|-----------|---------|
| Equality | `attributes.type = "order"` |
| Not equal | `attributes.type != "test"` |
| Has attribute | `hasPrefix(attributes.key, "val")` |
| AND | `attributes.a = "x" AND attributes.b = "y"` |
| OR | `attributes.a = "x" OR attributes.a = "y"` |
| NOT | `NOT attributes.type = "test"` |

---

## Part 7 — Dead-Letter Topics

### Dead-Letter Queue (DLQ)

```
┌──────────────────────────────────────────────────────────────┐
│         DEAD-LETTER TOPICS                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Messages that fail processing after N attempts are sent     │
│  to a dead-letter topic instead of being retried forever:    │
│                                                                │
│  ┌───────────┐    ┌──────────────┐                           │
│  │   Topic   │───►│ Subscription │                           │
│  └───────────┘    │              │                           │
│                   │  attempt 1 ──► NACK (fail)               │
│                   │  attempt 2 ──► NACK (fail)               │
│                   │  attempt 3 ──► NACK (fail)               │
│                   │  attempt 4 ──► NACK (fail)               │
│                   │  attempt 5 ──► NACK (fail)               │
│                   │      │                                    │
│                   │      ▼ max attempts reached              │
│                   │  ┌──────────────────┐                    │
│                   │  │ Dead-Letter Topic │                    │
│                   │  │ (for inspection   │                    │
│                   │  │  and debugging)   │                    │
│                   │  └──────────────────┘                    │
│                   └──────────────────────┘                    │
│                                                                │
│  The dead-letter topic is a regular Pub/Sub topic.           │
│  Create a subscription on it to inspect failed messages.     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create dead-letter topic
gcloud pubsub topics create orders-dlq

# Create subscription for DLQ monitoring
gcloud pubsub subscriptions create orders-dlq-sub \
    --topic=orders-dlq

# Create main subscription with dead-letter policy
gcloud pubsub subscriptions create orders-sub \
    --topic=orders-topic \
    --dead-letter-topic=orders-dlq \
    --max-delivery-attempts=5

# Grant required permissions for DLQ forwarding
gcloud pubsub topics add-iam-policy-binding orders-dlq \
    --member="serviceAccount:service-PROJECT_NUM@gcp-sa-pubsub.iam.gserviceaccount.com" \
    --role="roles/pubsub.publisher"

gcloud pubsub subscriptions add-iam-policy-binding orders-sub \
    --member="serviceAccount:service-PROJECT_NUM@gcp-sa-pubsub.iam.gserviceaccount.com" \
    --role="roles/pubsub.subscriber"
```

---

## Part 8 — Schemas & Message Validation

### Schema Registry

```
┌──────────────────────────────────────────────────────────────┐
│         SCHEMAS                                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Define schemas to validate messages before publishing:       │
│                                                                │
│  Publisher ──► Topic (schema attached) ──► Validation         │
│                                               │               │
│                                       ┌───────┴──────┐       │
│                                       │              │       │
│                                    Valid          Invalid     │
│                                    → publish     → reject    │
│                                                                │
│  Supported formats:                                            │
│  • Apache Avro                                                │
│  • Protocol Buffers (Protobuf)                               │
│                                                                │
│  Message encoding:                                             │
│  • JSON                                                       │
│  • BINARY                                                     │
│                                                                │
│  Schema versions:                                              │
│  • Schemas support revisions (versioning)                    │
│  • Set first/last revision on topic to control range         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create an Avro schema
gcloud pubsub schemas create order-schema \
    --type=avro \
    --definition='{
      "type": "record",
      "name": "Order",
      "fields": [
        {"name": "orderId", "type": "string"},
        {"name": "amount", "type": "double"},
        {"name": "currency", "type": "string"},
        {"name": "timestamp", "type": "string"}
      ]
    }'

# Create a Protobuf schema
gcloud pubsub schemas create event-schema \
    --type=protocol-buffer \
    --definition-file=event.proto

# Validate a schema
gcloud pubsub schemas validate-schema \
    --type=avro \
    --definition-file=order.avsc

# Attach schema to topic
gcloud pubsub topics create validated-orders \
    --schema=order-schema \
    --message-encoding=json

# Validate a message against a schema
gcloud pubsub schemas validate-message \
    --message-encoding=json \
    --message='{"orderId":"123","amount":99.99,"currency":"USD","timestamp":"2024-01-15T10:00:00Z"}' \
    --schema-name=order-schema

# List schemas
gcloud pubsub schemas list

# Create a schema revision
gcloud pubsub schemas commit order-schema \
    --type=avro \
    --definition-file=order-v2.avsc
```

---

## Part 9 — Exactly-Once Delivery

### Exactly-Once vs At-Least-Once

```
┌──────────────────────────────────────────────────────────────┐
│         EXACTLY-ONCE DELIVERY                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Default (at-least-once):                                      │
│  → Messages may be delivered multiple times                  │
│  → Subscriber must handle duplicates (idempotent processing) │
│                                                                │
│  Exactly-once delivery:                                        │
│  → Each message delivered and acknowledged exactly once      │
│  → Uses acknowledgement IDs with server-side dedup           │
│  → Only for PULL subscriptions (not push)                    │
│  → Slightly higher latency                                   │
│                                                                │
│  Enable exactly-once:                                          │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  gcloud pubsub subscriptions create exact-sub \      │    │
│  │      --topic=orders-topic \                           │    │
│  │      --enable-exactly-once-delivery                   │    │
│  │                                                      │    │
│  │  Subscriber must:                                     │    │
│  │  • Use acknowledge with ack ID                       │    │
│  │  • Handle AcknowledgeConfirmation responses           │    │
│  │  • Retry failed acks                                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  When to use:                                                  │
│  • Financial transactions                                    │
│  • Inventory updates                                          │
│  • Any operation where duplicates cause data corruption      │
│                                                                │
│  When NOT to use:                                              │
│  • Logging / analytics (duplicates are tolerable)            │
│  • Notifications (idempotent by nature)                      │
│  • High-throughput streams (exactly-once adds overhead)      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 10 — BigQuery Subscriptions

### Direct Write to BigQuery

```
┌──────────────────────────────────────────────────────────────┐
│         BIGQUERY SUBSCRIPTIONS                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Write messages directly to a BigQuery table — no consumer   │
│  code needed:                                                  │
│                                                                │
│  Publisher ──► Topic ──► BigQuery Subscription ──► BQ Table  │
│                                                                │
│  Two modes:                                                    │
│  1. Write metadata: stores message data, attributes,         │
│     publish time, etc. in predefined columns                 │
│                                                                │
│  2. Use topic schema: maps schema fields to BQ columns       │
│     (requires schema attached to topic)                       │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  BigQuery table columns (metadata mode):             │    │
│  │  ├── subscription_name  STRING                       │    │
│  │  ├── message_id         STRING                       │    │
│  │  ├── publish_time       TIMESTAMP                    │    │
│  │  ├── data               BYTES or STRING              │    │
│  │  └── attributes         STRING (JSON)                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Benefits:                                                     │
│  • No subscriber code to write/maintain                      │
│  • Automatic retry and dead-lettering                        │
│  • Near real-time ingestion to BigQuery                      │
│  • Built-in deduplication                                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create BigQuery subscription
gcloud pubsub subscriptions create orders-bq-sub \
    --topic=orders-topic \
    --bigquery-table=my-project:dataset.orders_table \
    --write-metadata

# With topic schema mapping
gcloud pubsub subscriptions create orders-bq-sub \
    --topic=validated-orders \
    --bigquery-table=my-project:dataset.orders_table \
    --use-topic-schema

# With dead-letter topic for failed writes
gcloud pubsub subscriptions create orders-bq-sub \
    --topic=orders-topic \
    --bigquery-table=my-project:dataset.orders_table \
    --write-metadata \
    --dead-letter-topic=orders-dlq \
    --max-delivery-attempts=5
```

---

## Part 11 — Cloud Storage Subscriptions

### Direct Write to Cloud Storage

```
┌──────────────────────────────────────────────────────────────┐
│         CLOUD STORAGE SUBSCRIPTIONS                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Write messages directly to Cloud Storage as files:           │
│                                                                │
│  Publisher ──► Topic ──► GCS Subscription ──► GCS Bucket     │
│                                                                │
│  Messages are batched into files:                              │
│  ├── Format: Avro or text                                    │
│  ├── Batched by time (max duration) or size (max bytes)      │
│  ├── File prefix configurable                                │
│  └── Automatic file naming with timestamps                   │
│                                                                │
│  Output path example:                                          │
│  gs://bucket/prefix/YYYY/MM/DD/HH/file-XXXX.avro            │
│                                                                │
│  Use cases:                                                    │
│  • Long-term message archival                                │
│  • Data lake ingestion                                        │
│  • Compliance / audit log storage                            │
│  • Batch processing input                                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Create Cloud Storage subscription
gcloud pubsub subscriptions create events-gcs-sub \
    --topic=events-topic \
    --cloud-storage-bucket=my-events-bucket \
    --cloud-storage-file-prefix=events/ \
    --cloud-storage-file-suffix=.avro \
    --cloud-storage-max-duration=5m \
    --cloud-storage-max-bytes=1000000000
```

---

## Part 12 — Retention, Replay & Snapshots

### Message Retention & Replay

```
┌──────────────────────────────────────────────────────────────┐
│         RETENTION, REPLAY & SNAPSHOTS                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Message Retention (topic level):                              │
│  → Keep messages for a set duration (up to 31 days)          │
│  → Enables replaying messages from a point in time           │
│                                                                │
│  Message Retention (subscription level):                       │
│  → retain-acked-messages = true                               │
│  → Keeps acknowledged messages for replay                    │
│  → Duration: up to 7 days                                    │
│                                                                │
│  Seek (replay):                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Seek to a timestamp:                                 │    │
│  │  → Replay all messages published after that time     │    │
│  │  → Previously acked messages get redelivered          │    │
│  │                                                      │    │
│  │  Seek to a snapshot:                                  │    │
│  │  → Restore subscription state to snapshot point      │    │
│  │  → Messages acked after snapshot are redelivered     │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Snapshots:                                                    │
│  → Captures subscription's ack state at a point in time     │
│  → Used for safe deployments (rollback consumer state)      │
│  → Retained for 7 days (or topic retention, whichever less) │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Enable topic-level retention
gcloud pubsub topics update orders-topic \
    --message-retention-duration=3d

# Enable subscription-level retention
gcloud pubsub subscriptions update orders-sub \
    --retain-acked-messages \
    --message-retention-duration=7d

# Create a snapshot
gcloud pubsub snapshots create orders-snapshot \
    --subscription=orders-sub

# Seek to a snapshot (replay)
gcloud pubsub subscriptions seek orders-sub \
    --snapshot=orders-snapshot

# Seek to a timestamp (replay from a point in time)
gcloud pubsub subscriptions seek orders-sub \
    --time="2024-01-15T10:00:00Z"

# List snapshots
gcloud pubsub snapshots list
```

---

## Part 13 — IAM & Security

### IAM Roles

| Role | Description |
|------|-------------|
| `roles/pubsub.admin` | Full control over topics and subscriptions |
| `roles/pubsub.editor` | Create/update/delete topics and subscriptions |
| `roles/pubsub.viewer` | View topics, subscriptions, and messages |
| `roles/pubsub.publisher` | Publish messages to topics |
| `roles/pubsub.subscriber` | Consume messages from subscriptions |

### Security Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│         SECURITY BEST PRACTICES                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Use separate roles for publish vs subscribe              │
│     → Publisher gets roles/pubsub.publisher on topic        │
│     → Consumer gets roles/pubsub.subscriber on subscription │
│                                                                │
│  2. Grant per-resource IAM (not project-wide)                │
│     → Each service account → specific topics/subs only      │
│                                                                │
│  3. Enable CMEK for sensitive data                           │
│     → Encrypt messages at rest with your own KMS key        │
│                                                                │
│  4. Use push auth for push subscriptions                      │
│     → --push-auth-service-account for authenticated push    │
│     → Endpoint receives signed JWT for verification         │
│                                                                │
│  5. Use schemas to validate message format                    │
│     → Prevent malformed messages from entering the system   │
│                                                                │
│  6. Don't put sensitive data in message attributes           │
│     → Attributes are not encrypted separately               │
│     → Use message body (data field) for sensitive content   │
│                                                                │
│  7. Enable VPC Service Controls for Pub/Sub                  │
│     → Restrict API access to authorized networks            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Complete Setup

```hcl
# ─── Topic ────────────────────────────────────────────────────
resource "google_pubsub_topic" "orders" {
  name    = "orders-topic"
  project = var.project_id

  message_retention_duration = "604800s"  # 7 days

  labels = {
    environment = "production"
    team        = "orders"
  }
}

# ─── Topic with Schema ───────────────────────────────────────
resource "google_pubsub_schema" "order" {
  name       = "order-schema"
  project    = var.project_id
  type       = "AVRO"
  definition = jsonencode({
    type = "record"
    name = "Order"
    fields = [
      { name = "orderId",   type = "string" },
      { name = "amount",    type = "double" },
      { name = "currency",  type = "string" },
      { name = "timestamp", type = "string" }
    ]
  })
}

resource "google_pubsub_topic" "validated_orders" {
  name    = "validated-orders"
  project = var.project_id

  schema_settings {
    schema   = google_pubsub_schema.order.id
    encoding = "JSON"
  }
}

# ─── Topic with CMEK ─────────────────────────────────────────
resource "google_pubsub_topic" "secure" {
  name         = "secure-topic"
  project      = var.project_id
  kms_key_name = google_kms_crypto_key.pubsub.id
}

# ─── Dead-Letter Topic ───────────────────────────────────────
resource "google_pubsub_topic" "orders_dlq" {
  name    = "orders-dlq"
  project = var.project_id
}

# ─── Pull Subscription ───────────────────────────────────────
resource "google_pubsub_subscription" "orders_pull" {
  name    = "orders-pull-sub"
  project = var.project_id
  topic   = google_pubsub_topic.orders.id

  ack_deadline_seconds       = 60
  message_retention_duration = "604800s"  # 7 days
  retain_acked_messages      = true
  enable_message_ordering    = true

  retry_policy {
    minimum_backoff = "10s"
    maximum_backoff = "600s"
  }

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.orders_dlq.id
    max_delivery_attempts = 5
  }

  labels = {
    environment = "production"
  }
}

# ─── Push Subscription ───────────────────────────────────────
resource "google_pubsub_subscription" "orders_push" {
  name    = "orders-push-sub"
  project = var.project_id
  topic   = google_pubsub_topic.orders.id

  ack_deadline_seconds = 60

  push_config {
    push_endpoint = "https://my-service.run.app/pubsub"

    oidc_token {
      service_account_email = google_service_account.pubsub_invoker.email
      audience              = "https://my-service.run.app"
    }
  }
}

# ─── BigQuery Subscription ───────────────────────────────────
resource "google_pubsub_subscription" "orders_bq" {
  name    = "orders-bq-sub"
  project = var.project_id
  topic   = google_pubsub_topic.orders.id

  bigquery_config {
    table          = "${var.project_id}.orders_dataset.orders_raw"
    write_metadata = true
  }
}

# ─── Cloud Storage Subscription ──────────────────────────────
resource "google_pubsub_subscription" "events_gcs" {
  name    = "events-gcs-sub"
  project = var.project_id
  topic   = google_pubsub_topic.orders.id

  cloud_storage_config {
    bucket          = google_storage_bucket.events.name
    filename_prefix = "events/"
    filename_suffix = ".avro"
    max_duration    = "300s"
    max_bytes       = 1000000000

    avro_config {
      write_metadata = true
    }
  }
}

# ─── Exactly-Once Subscription ───────────────────────────────
resource "google_pubsub_subscription" "orders_exact" {
  name    = "orders-exact-sub"
  project = var.project_id
  topic   = google_pubsub_topic.orders.id

  ack_deadline_seconds           = 60
  enable_exactly_once_delivery   = true
}

# ─── IAM ──────────────────────────────────────────────────────
resource "google_pubsub_topic_iam_member" "publisher" {
  project = var.project_id
  topic   = google_pubsub_topic.orders.id
  role    = "roles/pubsub.publisher"
  member  = "serviceAccount:${google_service_account.order_service.email}"
}

resource "google_pubsub_subscription_iam_member" "subscriber" {
  project      = var.project_id
  subscription = google_pubsub_subscription.orders_pull.id
  role         = "roles/pubsub.subscriber"
  member       = "serviceAccount:${google_service_account.worker.email}"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# TOPICS
# ═══════════════════════════════════════════════════════════════
gcloud pubsub topics create TOPIC [--message-retention-duration=DURATION]
gcloud pubsub topics list
gcloud pubsub topics describe TOPIC
gcloud pubsub topics delete TOPIC
gcloud pubsub topics update TOPIC [--message-retention-duration=DURATION]
gcloud pubsub topics publish TOPIC --message=MSG [--attribute=K=V] [--ordering-key=KEY]

# ═══════════════════════════════════════════════════════════════
# SUBSCRIPTIONS
# ═══════════════════════════════════════════════════════════════
gcloud pubsub subscriptions create SUB --topic=TOPIC [options]
gcloud pubsub subscriptions list
gcloud pubsub subscriptions describe SUB
gcloud pubsub subscriptions delete SUB
gcloud pubsub subscriptions pull SUB [--auto-ack] [--limit=N]
gcloud pubsub subscriptions ack SUB --ack-ids=IDS
gcloud pubsub subscriptions seek SUB --time=TIMESTAMP | --snapshot=SNAP
gcloud pubsub subscriptions update SUB [options]

# ═══════════════════════════════════════════════════════════════
# SCHEMAS
# ═══════════════════════════════════════════════════════════════
gcloud pubsub schemas create SCHEMA --type=avro|protocol-buffer --definition=DEF
gcloud pubsub schemas list
gcloud pubsub schemas describe SCHEMA
gcloud pubsub schemas delete SCHEMA
gcloud pubsub schemas validate-schema --type=TYPE --definition-file=FILE
gcloud pubsub schemas validate-message --schema-name=S --message=M --message-encoding=E

# ═══════════════════════════════════════════════════════════════
# SNAPSHOTS
# ═══════════════════════════════════════════════════════════════
gcloud pubsub snapshots create SNAP --subscription=SUB
gcloud pubsub snapshots list
gcloud pubsub snapshots delete SNAP
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Event-Driven Microservices

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: EVENT-DRIVEN ORDER PROCESSING                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────┐                                                       │
│  │ Order API  │──publish──► orders-topic                              │
│  │ (Cloud Run)│                  │                                    │
│  └────────────┘                  ├──► payment-sub ──► Payment Service │
│                                  ├──► inventory-sub ──► Inventory Svc │
│                                  ├──► notification-sub ──► Email Svc  │
│                                  └──► analytics-bq-sub ──► BigQuery  │
│                                                                        │
│  Each service subscribes independently:                               │
│  • Payment: pull sub with ordering (per order ID)                    │
│  • Inventory: pull sub with exactly-once delivery                    │
│  • Notifications: push sub → Cloud Functions                         │
│  • Analytics: BigQuery sub (no code needed)                          │
│                                                                        │
│  Dead-letter: Each sub has its own DLQ                                │
│  Schema: Avro schema validates order format                          │
│  Retry: exponential backoff (10s → 600s)                             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Log Aggregation Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: CENTRALIZED LOG AGGREGATION                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Service A ──┐                                                        │
│  Service B ──┼──publish──► logs-topic                                │
│  Service C ──┘                  │                                     │
│                                  ├──► logs-bq-sub ──► BigQuery       │
│                                  │    (all logs for analytics)       │
│                                  │                                    │
│                                  ├──► logs-gcs-sub ──► GCS           │
│                                  │    (long-term archival, Avro)     │
│                                  │                                    │
│                                  └──► errors-sub ──► Cloud Function  │
│                                       filter: attributes.level       │
│                                         = "ERROR"                     │
│                                       → sends PagerDuty alert        │
│                                                                        │
│  Topic retention: 7 days (enables replay for debugging)              │
│  GCS sub: batches every 5 min or 1 GB                                │
│  Error sub: filtered to only receive ERROR-level logs                │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Safe Deployment with Snapshots

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: SAFE CONSUMER DEPLOYMENT                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Problem: Deploying a new consumer version that might have bugs      │
│                                                                        │
│  Step 1: Create snapshot before deployment                            │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  gcloud pubsub snapshots create pre-deploy-snap \        │        │
│  │      --subscription=orders-sub                           │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Step 2: Deploy new consumer version                                  │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  gcloud run deploy order-processor --image=v2.0          │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Step 3: Monitor for errors...                                        │
│                                                                        │
│  If bugs found — rollback consumer AND replay messages:              │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  gcloud run deploy order-processor --image=v1.9          │        │
│  │  gcloud pubsub subscriptions seek orders-sub \           │        │
│  │      --snapshot=pre-deploy-snap                           │        │
│  │                                                           │        │
│  │  → Subscription resets to pre-deployment state           │        │
│  │  → Messages processed by buggy v2.0 are redelivered     │        │
│  │  → v1.9 reprocesses them correctly                       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create topic | `gcloud pubsub topics create TOPIC` |
| Publish message | `gcloud pubsub topics publish TOPIC --message=MSG` |
| Create pull sub | `gcloud pubsub subscriptions create SUB --topic=T` |
| Create push sub | `... --push-endpoint=URL --push-auth-service-account=SA` |
| Pull messages | `gcloud pubsub subscriptions pull SUB --auto-ack` |
| Enable ordering | `--enable-message-ordering` (sub) + `--ordering-key=K` (publish) |
| Add filter | `--filter='attributes.type = "order"'` |
| Dead-letter | `--dead-letter-topic=DLQ --max-delivery-attempts=5` |
| BigQuery sub | `--bigquery-table=P:D.T --write-metadata` |
| GCS sub | `--cloud-storage-bucket=B` |
| Exactly-once | `--enable-exactly-once-delivery` |
| Create snapshot | `gcloud pubsub snapshots create S --subscription=SUB` |
| Seek (replay) | `gcloud pubsub subscriptions seek SUB --time=T` |
| Max message size | 10 MiB |
| Max retention | 31 days (topic), 7 days (subscription) |
| Free tier | 10 GiB/month |

---

## What is Pub/Sub? (Beginner Explanation)

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUB/SUB — THE SIMPLE EXPLANATION                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 💡 Pub/Sub is like a newspaper delivery system:                      │
│    • Publishers = newspapers/magazines that create content           │
│    • Topics = the newspaper titles (e.g., "Daily News")             │
│    • Subscriptions = delivery routes / mailbox slots                 │
│    • Subscribers = people who read the newspapers                    │
│    • Pub/Sub = the delivery company in between                       │
│                                                                       │
│ The publisher doesn't need to know who's reading.                    │
│ The subscriber doesn't need to know who's writing.                   │
│ Pub/Sub handles ALL the delivery.                                    │
│                                                                       │
│ Why Decoupling Matters                                               │
│ ──────────────────────                                               │
│                                                                       │
│ Without Pub/Sub (tightly coupled):                                   │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Order Service must know about:                                │    │
│ │ ├── Payment Service URL → calls it directly                  │    │
│ │ ├── Inventory Service URL → calls it directly                │    │
│ │ ├── Email Service URL → calls it directly                    │    │
│ │ └── Analytics Service URL → calls it directly                │    │
│ │                                                                │    │
│ │ Problems:                                                      │    │
│ │ ├── If Payment Service is down → Order Service breaks        │    │
│ │ ├── Adding a new service → change Order Service code         │    │
│ │ ├── Each service must handle retries itself                   │    │
│ │ └── One slow service slows down everything                   │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ With Pub/Sub (decoupled):                                            │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Order Service publishes to "orders-topic" → done!            │    │
│ │                                                                │    │
│ │ Each service subscribes independently:                        │    │
│ │ ├── Payment Service → subscribes, processes at its pace      │    │
│ │ ├── Inventory Service → subscribes, processes at its pace    │    │
│ │ ├── Email Service → subscribes, processes at its pace        │    │
│ │ └── Analytics Service → subscribes, writes to BigQuery       │    │
│ │                                                                │    │
│ │ Benefits:                                                      │    │
│ │ ├── If Payment is down → messages wait, nothing breaks       │    │
│ │ ├── Adding a new service → just create a new subscription    │    │
│ │ ├── Pub/Sub handles retries and dead-lettering               │    │
│ │ └── Services scale independently                              │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ When to use Pub/Sub vs Cloud Tasks                                   │
│ ──────────────────────────────────                                   │
│                                                                       │
│ ┌────────────────┬──────────────────────┬──────────────────────┐    │
│ │                │ Pub/Sub              │ Cloud Tasks           │    │
│ ├────────────────┼──────────────────────┼──────────────────────┤    │
│ │ Pattern        │ Fan-out to many      │ One specific target  │    │
│ │                │ subscribers          │                       │    │
│ │ Who picks the  │ Subscriber decides   │ Creator decides      │    │
│ │ target?        │ (subscribe to topic) │ (sets target URL)    │    │
│ │ Delivery       │ Immediate (real-time)│ Can schedule / delay │    │
│ │ Rate control   │ No (fire & forget)   │ Yes (rate limiting)  │    │
│ │ Deduplication  │ At-least-once or     │ At-least-once        │    │
│ │                │ exactly-once         │                       │    │
│ │ Use case       │ Events, streaming,   │ Background jobs,     │    │
│ │                │ notifications        │ deferred work        │    │
│ └────────────────┴──────────────────────┴──────────────────────┘    │
│                                                                       │
│ Rule of thumb:                                                       │
│ → "Something happened, whoever cares can react" → Pub/Sub          │
│ → "Do this specific task later" → Cloud Tasks                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Creating & Managing Topics

```
┌─────────────────────────────────────────────────────────────────────┐
│           CREATING A TOPIC FROM CONSOLE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Pub/Sub → Topics → [+ Create Topic]                     │
│                                                                       │
│ ── Fields ──                                                        │
│                                                                       │
│ Topic ID:                                                            │
│ ├── Name for your topic (e.g., "orders-topic")                     │
│ ├── Must be unique within the project                               │
│ ├── Allowed: letters, numbers, hyphens, underscores, periods, ~    │
│ ├── 3-255 characters                                                │
│ └── Full path: projects/PROJECT_ID/topics/TOPIC_ID                 │
│                                                                       │
│ Add a default subscription:                                         │
│ ├── ☑ Checked by default                                           │
│ ├── Auto-creates a pull subscription named "TOPIC_ID-sub"          │
│ └── You can uncheck this and create subscriptions later             │
│                                                                       │
│ Schema:                                                              │
│ ├── Optional — validates message format before publishing           │
│ ├── Select an existing schema or leave as "None"                    │
│ ├── If selected, choose message encoding: JSON or BINARY            │
│ └── Messages that don't match schema are rejected at publish        │
│                                                                       │
│ Message retention duration:                                          │
│ ├── Default: not set (messages kept only until acknowledged)        │
│ ├── Range: 10 minutes to 31 days                                   │
│ ├── Enables message replay (seek to a timestamp)                    │
│ └── ⚡ Retained messages incur storage costs ($0.27/GiB/month)       │
│                                                                       │
│ Encryption:                                                          │
│ ├── Google-managed encryption key (default) — no setup needed       │
│ └── Customer-managed encryption key (CMEK) — select your KMS key   │
│                                                                       │
│ Labels:                                                              │
│ ├── Optional key-value pairs (e.g., env=prod, team=orders)         │
│ └── Useful for cost tracking and filtering                          │
│                                                                       │
│ Click [Create] → topic is ready to receive messages                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           CREATING A SUBSCRIPTION FROM CONSOLE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Pub/Sub → Subscriptions → [+ Create Subscription]       │
│                                                                       │
│ ── Basic Settings ──                                                │
│                                                                       │
│ Subscription ID:                                                     │
│ ├── Name for your subscription (e.g., "orders-pull-sub")           │
│ └── Must be unique within the project                               │
│                                                                       │
│ Select a Cloud Pub/Sub topic:                                       │
│ ├── Choose the topic this subscription attaches to                  │
│ └── One topic can have many subscriptions (fan-out)                 │
│                                                                       │
│ Delivery type:                                                       │
│ ├── Pull (default):                                                 │
│ │   ├── Your application pulls messages when ready                 │
│ │   ├── Best for: high throughput, batch processing, workers       │
│ │   └── No endpoint needed — subscriber calls Pub/Sub API          │
│ │                                                                    │
│ └── Push:                                                            │
│     ├── Pub/Sub sends messages to your HTTP(S) endpoint            │
│     ├── Endpoint URL: must be HTTPS (e.g., Cloud Run URL)          │
│     ├── Authentication:                                              │
│     │   ├── Enable authentication (recommended)                    │
│     │   ├── Service account: SA that signs the push JWT            │
│     │   └── Audience: your endpoint URL                            │
│     └── Best for: Cloud Run, Cloud Functions, serverless           │
│                                                                       │
│ ── Message Delivery Settings ──                                     │
│                                                                       │
│ Acknowledgement deadline:                                            │
│ ├── How long a subscriber has to ack before message is redelivered │
│ ├── Default: 10 seconds                                             │
│ ├── Range: 10 to 600 seconds (10 minutes)                          │
│ └── ⚡ Set higher for slow-processing subscribers                    │
│                                                                       │
│ Message retention duration:                                          │
│ ├── How long unacked messages are kept                               │
│ ├── Default: 7 days                                                  │
│ └── Range: 10 minutes to 7 days                                     │
│                                                                       │
│ Retain acknowledged messages:                                        │
│ ├── ☐ Off by default                                                │
│ ├── When on: keeps acked messages for replay (seek)                 │
│ └── ⚡ Incurs storage costs for retained messages                     │
│                                                                       │
│ Message ordering:                                                    │
│ ├── ☐ Off by default                                                │
│ ├── When on: delivers messages in publish order (per ordering key) │
│ └── Publisher must set ordering key on messages                      │
│                                                                       │
│ Exactly once delivery:                                               │
│ ├── ☐ Off by default                                                │
│ ├── When on: guarantees each message processed exactly once         │
│ └── Only available for pull subscriptions                            │
│                                                                       │
│ ── Dead Lettering ──                                                │
│                                                                       │
│ Dead-letter topic:                                                   │
│ ├── Select a topic where failed messages are sent                   │
│ ├── Max delivery attempts: 5 (default) — range 5 to 100            │
│ └── ⚡ Requires Pub/Sub service account to have publisher role       │
│     on the dead-letter topic                                         │
│                                                                       │
│ ── Retry Policy ──                                                  │
│                                                                       │
│ Retry after nack:                                                    │
│ ├── Minimum backoff: 10 seconds (default)                           │
│ └── Maximum backoff: 600 seconds (default)                          │
│                                                                       │
│ ── Filter ──                                                        │
│                                                                       │
│ Subscription filter:                                                 │
│ ├── Optional — only receive messages matching this filter           │
│ ├── Syntax: attributes.type = "order"                               │
│ ├── ⚡ Filtered-out messages are auto-acknowledged (you don't pay    │
│ │   for delivery but you pay for ingestion)                         │
│ └── ⚠️ Cannot change filter after creation — must recreate          │
│                                                                       │
│ Click [Create] → subscription is active and ready to receive        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PUBLISHING A TEST MESSAGE FROM CONSOLE                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Pub/Sub → Topics → Click your topic name                 │
│                                                                       │
│ At the top of the topic detail page → [Messages] tab               │
│ → Click [Publish Message]                                           │
│                                                                       │
│ Message body:                                                        │
│ ├── Enter your message text or JSON                                 │
│ ├── Example: {"orderId": "12345", "amount": 99.99}                 │
│ └── This becomes the "data" field (base64-encoded internally)       │
│                                                                       │
│ Message attributes (optional):                                       │
│ ├── Click [+ Add attribute]                                         │
│ ├── Key: type        Value: order                                   │
│ ├── Key: priority    Value: high                                    │
│ └── Attributes are key-value metadata (used for filtering)          │
│                                                                       │
│ Ordering key (optional):                                             │
│ ├── Only relevant if subscriptions have ordering enabled            │
│ └── Enter a key like "customer-123" to group ordered messages       │
│                                                                       │
│ Click [Publish] → message is published to the topic                 │
│ → ✅ You'll see a confirmation with the message ID                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PULLING MESSAGES FROM CONSOLE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Pub/Sub → Subscriptions → Click your subscription        │
│                                                                       │
│ → [Messages] tab → Click [Pull]                                     │
│                                                                       │
│ The console pulls pending messages and displays them:               │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Message ID   │ Data (decoded)         │ Attributes │ Time   │    │
│ ├──────────────┼────────────────────────┼────────────┼────────┤    │
│ │ 12345678     │ {"orderId": "12345"..} │ type=order │ 10:30  │    │
│ └──────────────┴────────────────────────┴────────────┴────────┘    │
│                                                                       │
│ Enable auto-acknowledge:                                             │
│ ├── ☑ Toggle "Enable ack messages" before pulling                   │
│ ├── When on: messages are auto-acknowledged after pull              │
│ └── When off: messages are NOT acked — they will be redelivered     │
│     after the ack deadline expires                                   │
│                                                                       │
│ ⚡ Console pull is for debugging/testing only.                        │
│   In production, use client libraries or gcloud CLI.                │
│                                                                       │
│ ⚡ If no messages appear:                                             │
│   ├── Ensure messages were published AFTER the subscription was     │
│   │   created (subscriptions don't get retroactive messages)        │
│   ├── Check if a filter is excluding your messages                  │
│   └── Check if another subscriber already acked the messages        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DELETING A SUBSCRIPTION FROM CONSOLE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Pub/Sub → Subscriptions                                  │
│                                                                       │
│ Option 1: From the list                                              │
│ ├── ☑ Check the box next to the subscription name                   │
│ ├── Click [Delete] in the top action bar                            │
│ └── Confirm deletion in the popup                                    │
│                                                                       │
│ Option 2: From the subscription detail page                         │
│ ├── Click the subscription name to open it                          │
│ ├── Click [Delete] at the top of the page                           │
│ └── Confirm deletion                                                 │
│                                                                       │
│ ⚠️ Deletion is permanent!                                             │
│ ├── All unacknowledged messages in this subscription are lost       │
│ ├── The subscription name becomes available for reuse               │
│ ├── Other subscriptions on the same topic are NOT affected          │
│ └── The topic itself is NOT affected                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DELETING A TOPIC FROM CONSOLE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Pub/Sub → Topics                                         │
│                                                                       │
│ Option 1: From the list                                              │
│ ├── ☑ Check the box next to the topic name                          │
│ ├── Click [Delete] in the top action bar                            │
│ └── Confirm deletion in the popup                                    │
│                                                                       │
│ Option 2: From the topic detail page                                 │
│ ├── Click the topic name to open it                                  │
│ ├── Click [Delete] at the top of the page                           │
│ └── Confirm deletion                                                 │
│                                                                       │
│ ⚠️ Deletion is permanent!                                             │
│ ├── All subscriptions attached to this topic are also deleted       │
│ ├── Any publishers still sending to this topic will get errors      │
│ ├── The topic name becomes available for reuse                      │
│ └── ⚡ If this topic is used as a dead-letter topic for other        │
│     subscriptions, those subscriptions will lose their DLQ —        │
│     failed messages will be retried indefinitely instead.           │
│                                                                       │
│ CLI equivalent:                                                      │
│ gcloud pubsub topics delete orders-topic                             │
│ # This also deletes all subscriptions attached to the topic         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Monitoring & Troubleshooting

```
┌─────────────────────────────────────────────────────────────────────┐
│           KEY METRICS TO WATCH                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. oldest_unacked_message_age (subscription)                        │
│    ├── How old is the oldest unprocessed message (in seconds)       │
│    ├── Healthy: low value (seconds to minutes)                      │
│    ├── ⚠️ High value = subscriber is falling behind                  │
│    └── Alert threshold: set based on your SLA (e.g., > 300s)       │
│                                                                       │
│ 2. num_undelivered_messages (subscription)                           │
│    ├── Count of messages waiting to be delivered                     │
│    ├── Also called "backlog"                                         │
│    ├── Healthy: stays near zero or fluctuates within a range        │
│    └── ⚠️ Continuously growing = subscriber can't keep up            │
│                                                                       │
│ 3. publish_message_count (topic)                                     │
│    ├── Number of messages published to the topic                     │
│    └── Useful for tracking publish rate and detecting spikes        │
│                                                                       │
│ 4. pull_request_count / push_request_count (subscription)           │
│    ├── How often subscribers are pulling / receiving pushes          │
│    └── Low value = subscriber may be down or misconfigured          │
│                                                                       │
│ 5. dead_letter_message_count (subscription)                          │
│    ├── Messages sent to dead-letter topic                            │
│    └── ⚠️ High value = messages are consistently failing             │
│                                                                       │
│ 6. ack_latency (subscription)                                        │
│    ├── Time between delivery and acknowledgement                     │
│    └── High latency = processing is slow                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           COMMON ISSUES & SOLUTIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Messages are not being received                             │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Check:                                                        │    │
│ │ ├── Was the subscription created BEFORE messages were         │    │
│ │ │   published? Subscriptions only get messages published     │    │
│ │ │   AFTER their creation.                                     │    │
│ │ ├── Is there a filter on the subscription that excludes      │    │
│ │ │   your messages? Check attributes match the filter.        │    │
│ │ ├── Is another consumer acking the messages first?           │    │
│ │ │   Within a subscription, each message goes to ONE consumer.│    │
│ │ ├── For push: is the endpoint returning 200 OK?              │    │
│ │ │   Check Cloud Logging for push delivery errors.            │    │
│ │ └── For push: does the Pub/Sub SA have invoker role on the  │    │
│ │     endpoint (e.g., roles/run.invoker for Cloud Run)?        │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Problem: Subscription backlog is growing                             │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Check:                                                        │    │
│ │ ├── Is the subscriber running? Check if pull requests are    │    │
│ │ │   being made (pull_request_count metric).                  │    │
│ │ ├── Is the subscriber processing too slowly?                 │    │
│ │ │   → Scale up consumers (more instances / workers)         │    │
│ │ │   → Increase flow control max_messages                    │    │
│ │ ├── Is the ack deadline too short?                           │    │
│ │ │   → Messages timeout and get redelivered, creating a loop │    │
│ │ │   → Increase ack deadline or use modifyAckDeadline        │    │
│ │ └── Is the publisher sending a burst of messages?            │    │
│ │     → Backlog may be temporary, monitor if it stabilizes    │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Problem: Messages being delivered multiple times                     │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Check:                                                        │    │
│ │ ├── Is the subscriber acking within the ack deadline?        │    │
│ │ │   → If processing takes longer than deadline, message is  │    │
│ │ │     redelivered. Increase ack deadline.                    │    │
│ │ ├── At-least-once delivery is the default — duplicates are  │    │
│ │ │   expected! Make your processing idempotent.               │    │
│ │ └── Need exactly-once? Enable exactly-once delivery on the  │    │
│ │     subscription (pull mode only).                           │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Problem: Dead-letter messages piling up                              │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Check:                                                        │    │
│ │ ├── Pull messages from the DLQ subscription to inspect them  │    │
│ │ ├── Check message attributes for delivery attempt count      │    │
│ │ ├── Common causes:                                            │    │
│ │ │   ├── Malformed message data (schema mismatch)             │    │
│ │ │   ├── Subscriber bug that always fails on certain messages │    │
│ │ │   └── Downstream dependency failures (DB down, API errors) │    │
│ │ └── Fix the root cause, then republish DLQ messages to the  │    │
│ │     original topic for reprocessing                           │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           VIEWING METRICS IN CONSOLE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Option 1: From the Pub/Sub Console                                  │
│ ├── Console → Pub/Sub → Subscriptions → Click subscription         │
│ ├── [Monitoring] tab                                                │
│ ├── Built-in charts:                                                │
│ │   ├── Unacked message count (backlog)                            │
│ │   ├── Oldest unacked message age                                  │
│ │   ├── Pull/push request count                                    │
│ │   └── Ack message count                                          │
│ └── Adjust time range (1h, 6h, 1d, 7d, 30d)                       │
│                                                                       │
│ Option 2: From Cloud Monitoring                                     │
│ ├── Console → Monitoring → Metrics Explorer                        │
│ ├── Resource type: Cloud Pub/Sub Subscription                       │
│ ├── Metric: select from available Pub/Sub metrics                   │
│ │   ├── subscription/oldest_unacked_message_age                    │
│ │   ├── subscription/num_undelivered_messages                      │
│ │   ├── subscription/dead_letter_message_count                     │
│ │   └── subscription/ack_latencies                                 │
│ ├── Filter by subscription name if needed                           │
│ └── ⚡ Create alerting policies directly from Metrics Explorer:      │
│     → Click [Create Alert] → set threshold → add notification      │
│     → Example: Alert when oldest_unacked_message_age > 300s        │
│                                                                       │
│ Option 3: From the Topic page                                       │
│ ├── Console → Pub/Sub → Topics → Click topic                      │
│ ├── [Monitoring] tab                                                │
│ └── Shows publish rate, message size, and error rate                │
│                                                                       │
│ Recommended alerts to set up:                                       │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Alert                          │ Threshold          │ Why    │    │
│ ├────────────────────────────────┼────────────────────┼────────┤    │
│ │ oldest_unacked_message_age     │ > 300s (5 min)     │ Backlog│    │
│ │ num_undelivered_messages       │ > 10,000           │ Pileup │    │
│ │ dead_letter_message_count      │ > 0 (per hour)     │ Failures│   │
│ │ push error rate                │ > 5%               │ Endpoint│   │
│ └────────────────────────────────┴────────────────────┴────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 47: Cloud Tasks** → `47-cloud-tasks.md`
