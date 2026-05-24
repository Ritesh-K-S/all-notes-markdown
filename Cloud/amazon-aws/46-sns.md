# Chapter 46: SNS (Simple Notification Service)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: SNS Fundamentals](#part-1-sns-fundamentals)
- [Part 2: Creating a Topic (Full Portal Walkthrough)](#part-2-creating-a-topic-full-portal-walkthrough)
- [Part 3: Creating Subscriptions (Full Portal Walkthrough)](#part-3-creating-subscriptions-full-portal-walkthrough)
- [Part 4: Message Filtering & Attributes](#part-4-message-filtering--attributes)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Pub/Sub? Why Do We Need SNS?

Imagine a radio station. The station **broadcasts** (publishes) music, and anyone who tunes in (subscribes) hears it. The station doesn't need to know who's listening — it just sends the signal.

**SNS (Simple Notification Service) works the same way:**
- A **publisher** sends a message to a **topic** (the radio station)
- All **subscribers** (listeners) to that topic receive the message instantly
- Subscribers can be: email addresses, phone numbers (SMS), Lambda functions, SQS queues, HTTP endpoints, or mobile push notifications

**SQS vs SNS — What's the difference?**
- **SQS**: One message → one consumer picks it up (like a to-do list)
- **SNS**: One message → ALL subscribers get it simultaneously (like a broadcast)
- **Common pattern (fan-out)**: SNS broadcasts to multiple SQS queues, each processing the message differently

**Simple real-world examples:**
- 🚨 CloudWatch alarm triggers → SNS sends email + SMS + Slack notification
- 🛨️ Order placed → SNS broadcasts to: email confirmation queue + inventory queue + analytics queue
- 📱 Mobile app sends push notifications to millions of devices via SNS

Amazon SNS is a fully managed pub/sub messaging service for application-to-application (A2A) and application-to-person (A2P) communication. Publishers send messages to topics, and subscribers receive them instantly.

```
What you'll learn:
├── SNS fundamentals (topics, subscriptions, pub/sub)
├── Creating a topic (portal walkthrough)
├── Creating subscriptions (portal walkthrough)
├── Message filtering (attribute-based routing)
├── SNS + SQS fan-out pattern
└── Terraform & CLI examples
```

---

## Part 1: SNS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW SNS WORKS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Publisher → Topic → Subscribers (fan-out)                           │
│                                                                       │
│                     ┌──────────┐                                    │
│                     │  Topic   │                                    │
│ ┌───────────┐      │          │     ┌─── SQS Queue 1             │
│ │ Publisher │─────▶│ order-   │────▶├─── SQS Queue 2             │
│ │           │      │ events   │     ├─── Lambda Function         │
│ └───────────┘      │          │     ├─── HTTP/HTTPS Endpoint     │
│                     └──────────┘     ├─── Email                    │
│                                      ├─── SMS                      │
│                                      └─── Kinesis Data Firehose   │
│                                                                       │
│ SNS vs SQS:                                                         │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ SNS           │ SQS                  │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Model                │ Pub/Sub       │ Queue (pull)         │   │
│ │ Delivery             │ Push (instant)│ Poll (consumer pulls)│   │
│ │ Consumers            │ Multiple      │ Single (per message) │   │
│ │ Persistence          │ No (instant)  │ Yes (retained)       │   │
│ │ Retry                │ Built-in      │ Visibility timeout   │   │
│ │ Use case             │ Broadcast     │ Decouple + buffer    │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
│ Topic types:                                                         │
│ ├── Standard: Unlimited throughput, best-effort ordering          │
│ └── FIFO: Strict ordering, deduplication, 300 publishes/s        │
│     → Name must end in .fifo                                      │
│     → Only SQS FIFO subscriptions supported                     │
│                                                                       │
│ Subscription protocols:                                              │
│ ├── SQS (⚡ most common for A2A)                                │
│ ├── Lambda                                                        │
│ ├── HTTP/HTTPS (webhook)                                         │
│ ├── Email / Email-JSON                                            │
│ ├── SMS                                                           │
│ ├── Kinesis Data Firehose                                        │
│ └── Platform application (mobile push: APNS, FCM)              │
│                                                                       │
│ Pricing:                                                             │
│ ├── First 1 million publishes/month: FREE                       │
│ ├── $0.50/million publishes after free tier                     │
│ ├── Deliveries: Free to SQS/Lambda, varies for SMS/email       │
│ └── SMS: $0.00645/message (US)                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Topic (Full Portal Walkthrough)

```
Console → SNS → Topics → Create topic

┌─────────────────────────────────────────────────────────────────┐
│           CREATE TOPIC - DETAILS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Type:                                                           │
│ ● Standard                                                     │
│   → Best-effort message ordering                             │
│   → ⚡ Supports all subscription protocols                   │
│                                                                   │
│ ○ FIFO                                                         │
│   → Strict ordering + deduplication                          │
│   → Only SQS FIFO subscriptions                              │
│   → Name must end in .fifo                                   │
│                                                                   │
│ Name: [order-events]                                          │
│ Display name: [Order Events] (optional, used in SMS/email)   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           ENCRYPTION                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Encryption: ☑ Enable encryption                               │
│                                                                   │
│ KMS key:                                                       │
│ ● alias/aws/sns (AWS managed, free)                           │
│ ○ Customer managed key: [alias/sns-key ▼]                    │
│   → ⚡ Required for cross-account access                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           ACCESS POLICY                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ● Basic                                                        │
│                                                                   │
│ Define who can publish messages to the topic:                 │
│ ● Only the topic owner                                        │
│ ○ Everyone                                                     │
│ ○ Only specified AWS accounts                                 │
│                                                                   │
│ Define who can subscribe to this topic:                       │
│ ● Only the topic owner                                        │
│ ○ Everyone                                                     │
│ ○ Only specified AWS accounts                                 │
│   → ⚡ Restrict for security                                 │
│                                                                   │
│ ○ Advanced (JSON policy)                                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           DELIVERY RETRY POLICY (HTTP/S)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Number of retries: [3]                                         │
│ Retry-backoff function: [Exponential]                         │
│ Min delay: [20] seconds                                       │
│ Max delay: [20] seconds                                       │
│ → Only applies to HTTP/HTTPS subscriptions                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           DELIVERY STATUS LOGGING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Log delivery status for protocols:                            │
│ ☑ Lambda                                                      │
│ ☑ SQS                                                         │
│ ☐ HTTP/S                                                      │
│ ☐ Application (mobile push)                                  │
│ ☐ Kinesis Data Firehose                                      │
│                                                                   │
│ IAM role: [SNSDeliveryLoggingRole ▼]                         │
│ Sample rate: [100] %                                          │
│ → Logs to CloudWatch Logs                                    │
│                                                                   │
│ Tags:                                                          │
│ [Environment] = [Production]                                  │
│                                                                   │
│                    [Create topic]                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating Subscriptions (Full Portal Walkthrough)

```
Console → SNS → Topics → order-events → Create subscription

┌─────────────────────────────────────────────────────────────────┐
│           CREATE SUBSCRIPTION                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Topic ARN: arn:aws:sns:us-east-1:123456:order-events          │
│ → Auto-filled if creating from topic page                    │
│                                                                   │
│ Protocol: [Amazon SQS ▼]                                      │
│ ├── Amazon SQS → Queue ARN                                   │
│ ├── AWS Lambda → Function ARN                                │
│ ├── HTTP → Endpoint URL                                      │
│ ├── HTTPS → Endpoint URL                                     │
│ ├── Email → Email address                                    │
│ ├── Email-JSON → Email address (raw JSON format)            │
│ ├── SMS → Phone number (+1234567890)                        │
│ └── Amazon Kinesis Data Firehose → Delivery stream ARN      │
│                                                                   │
│ Endpoint: [arn:aws:sqs:us-east-1:123456:order-processing]    │
│                                                                   │
│ ☑ Enable raw message delivery                                │
│ → Delivers message body only (no SNS metadata wrapper)      │
│ → ⚡ Enable for SQS/HTTP (simpler message format)           │
│ → Available for: SQS, HTTP/S, Kinesis Firehose              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           SUBSCRIPTION FILTER POLICY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ☑ Enable subscription filter policy                           │
│                                                                   │
│ Filter policy scope:                                           │
│ ● Message attributes (⚡ most common)                        │
│ ○ Message body                                                │
│                                                                   │
│ Filter policy (JSON):                                         │
│ {                                                              │
│   "order_type": ["premium", "express"],                      │
│   "amount": [{"numeric": [">=", 100]}]                       │
│ }                                                              │
│ → Only delivers messages matching this filter               │
│ → Reduces unnecessary processing                             │
│ → ⚡ Essential for targeted routing                          │
│                                                                   │
│ Filter operators:                                              │
│ ├── Exact match: ["value1", "value2"]                       │
│ ├── Prefix: [{"prefix": "order-"}]                           │
│ ├── Numeric: [{"numeric": [">=", 100]}]                      │
│ ├── Exists: [{"exists": true}]                               │
│ └── Anything-but: [{"anything-but": ["test"]}]               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           REDRIVE POLICY (DLQ for failed deliveries)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ☑ Enable redrive policy                                       │
│                                                                   │
│ Dead-letter queue:                                             │
│ [arn:aws:sqs:us-east-1:123456:sns-delivery-dlq ▼]           │
│ → SNS sends failed deliveries here after retries exhausted  │
│ → ⚡ Set up for HTTP/HTTPS and Lambda subscriptions          │
│                                                                   │
│                    [Create subscription]                       │
└─────────────────────────────────────────────────────────────────┘

Subscription confirmation:
├── SQS, Lambda: Auto-confirmed ✅
├── HTTP/HTTPS: Endpoint must confirm (GET/POST with token)
├── Email: Recipient must click confirmation link
└── SMS: Auto-confirmed ✅
```

---

## Part 4: Message Filtering & Attributes

```
┌─────────────────────────────────────────────────────────────────────┐
│           MESSAGE FILTERING IN ACTION                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Publish message with attributes:                                    │
│ {                                                                     │
│   "Message": "{\"orderId\":\"12345\",\"item\":\"widget\"}",       │
│   "MessageAttributes": {                                            │
│     "order_type": {                                                  │
│       "DataType": "String",                                         │
│       "StringValue": "premium"                                      │
│     },                                                                │
│     "amount": {                                                       │
│       "DataType": "Number",                                          │
│       "StringValue": "250"                                           │
│     }                                                                 │
│   }                                                                   │
│ }                                                                     │
│                                                                       │
│ Subscription filters:                                                │
│                                                                       │
│ Queue 1 (premium-orders):                                           │
│ Filter: {"order_type": ["premium"]}                                 │
│ → ✅ Receives this message                                          │
│                                                                       │
│ Queue 2 (standard-orders):                                           │
│ Filter: {"order_type": ["standard"]}                                │
│ → ❌ Does NOT receive this message                                  │
│                                                                       │
│ Queue 3 (high-value-orders):                                        │
│ Filter: {"amount": [{"numeric": [">=", 200]}]}                     │
│ → ✅ Receives this message (250 >= 200)                             │
│                                                                       │
│ Queue 4 (all-orders): No filter                                     │
│ → ✅ Receives ALL messages                                          │
│                                                                       │
│ ⚡ Filtering happens at SNS (no cost for filtered-out messages)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# Standard SNS topic
resource "aws_sns_topic" "orders" {
  name              = "order-events"
  kms_master_key_id = "alias/aws/sns"
}

# SQS subscription with filter
resource "aws_sns_topic_subscription" "premium_orders" {
  topic_arn            = aws_sns_topic.orders.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.premium.arn
  raw_message_delivery = true
  filter_policy = jsonencode({
    order_type = ["premium", "express"]
  })
}

# Lambda subscription
resource "aws_sns_topic_subscription" "analytics" {
  topic_arn = aws_sns_topic.orders.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.analytics.arn
}

# Email subscription
resource "aws_sns_topic_subscription" "alerts" {
  topic_arn = aws_sns_topic.orders.arn
  protocol  = "email"
  endpoint  = "alerts@example.com"
}

# FIFO topic
resource "aws_sns_topic" "fifo_orders" {
  name                        = "order-events.fifo"
  fifo_topic                  = true
  content_based_deduplication = true
}
```

```bash
# Create topic
aws sns create-topic --name order-events

# Subscribe SQS queue
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456:order-processing \
  --attributes '{"RawMessageDelivery":"true","FilterPolicy":"{\"order_type\":[\"premium\"]}"}'

# Publish message
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --message '{"orderId":"12345"}' \
  --message-attributes '{"order_type":{"DataType":"String","StringValue":"premium"}}'

# List subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Fan-out (SNS → Multiple SQS Queues)                    │
│ ┌──────┐    ┌──────┐    ┌── SQS: email-service                  │
│ │ API  │───▶│ SNS  │───▶├── SQS: inventory-service              │
│ │      │    │ Topic│    ├── SQS: analytics-service               │
│ └──────┘    └──────┘    └── SQS: audit-service                   │
│ → Each service processes independently                           │
│ → If one service is slow/down, others unaffected                │
│                                                                       │
│ Pattern 2: SNS + SQS + Lambda                                      │
│ S3 Event → SNS → SQS → Lambda (process images)                  │
│                 → SQS → Lambda (update database)                 │
│                 → SQS → Lambda (send notification)               │
│ → Each consumer has its own queue for buffering                 │
│                                                                       │
│ Pattern 3: Alert System                                              │
│ CloudWatch Alarm → SNS → Email (ops team)                        │
│                        → Lambda (auto-remediation)               │
│                        → HTTPS (PagerDuty/Slack webhook)        │
│                                                                       │
│ Pattern 4: Cross-Account Event Distribution                        │
│ Account A: SNS topic (with cross-account policy)                │
│ Account B: SQS subscription                                      │
│ Account C: Lambda subscription                                    │
│ → Centralized event bus for multi-account architecture          │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use SQS between SNS and Lambda for buffering                 │
│ 2. Enable raw message delivery for SQS subscriptions            │
│ 3. Use message filtering to reduce downstream processing       │
│ 4. Set up DLQ for failed deliveries                               │
│ 5. Encrypt topics with KMS                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
SNS Quick Reference:
├── Model: Pub/Sub (push-based, instant delivery)
├── Topics: Standard (unlimited throughput) or FIFO (300 pub/s)
├── Protocols: SQS, Lambda, HTTP/S, Email, SMS, Firehose, Mobile
├── Message size: Max 256 KB
├── Message filtering: Attribute-based (JSON filter policy)
├── Raw delivery: Enable for SQS (no SNS metadata wrapper)
├── FIFO: Name ends in .fifo, only SQS FIFO subscribers
├── Encryption: KMS (aws/sns key or CMK)
├── DLQ: SQS queue for failed SNS deliveries
├── Fan-out: SNS → multiple SQS queues (⚡ key pattern)
├── ⚡ Use message attributes + filter policies for routing
└── ⚡ SNS + SQS = reliable fan-out with buffering
```

---

## What's Next?

In **Chapter 47: EventBridge**, we'll cover event-driven architecture, event buses, rules, targets, and schema registry for building loosely coupled serverless applications.
