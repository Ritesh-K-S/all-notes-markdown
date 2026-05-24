# Chapter 45: SQS (Simple Queue Service)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: SQS Fundamentals](#part-1-sqs-fundamentals)
- [Part 2: Creating a Queue (Full Portal Walkthrough)](#part-2-creating-a-queue-full-portal-walkthrough)
- [Part 3: Message Operations](#part-3-message-operations)
- [Part 4: Dead-Letter Queues & Redrive](#part-4-dead-letter-queues--redrive)
- [Part 5: FIFO Queues](#part-5-fifo-queues)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is a Message Queue? Why Do We Need SQS?

Imagine a busy restaurant kitchen. When a waiter takes an order, they don't stand at the kitchen counter waiting for the food — they clip the **order ticket to a rail** and go serve other tables. The cooks pick up tickets and prepare food at their own pace.

**SQS is that order rail** for your applications. Instead of Service A calling Service B directly (and waiting/failing if B is busy), Service A drops a **message** in the queue and moves on. Service B picks up messages when it's ready.

**Why this matters ("decoupling"):**
- Without SQS: If the payment service is down, the order service crashes too
- With SQS: Orders pile up in the queue → when payment service comes back, it processes the backlog → no orders lost

**Simple real-world examples:**
- 🛨️ E-commerce: Order placed → message to queue → payment service processes when ready
- 📷 Image processing: Upload photo → message to queue → Lambda resizes when it picks up the message
- 📧 Email sending: User registers → message to queue → email service sends welcome email

Amazon SQS is a fully managed message queuing service that decouples producers from consumers. It provides reliable, scalable, asynchronous message delivery.

```
What you'll learn:
├── SQS fundamentals (queues, messages, polling)
├── Creating a Standard queue (portal walkthrough)
├── Creating a FIFO queue (portal walkthrough)
├── Message lifecycle (send, receive, delete, visibility)
├── Dead-letter queues & redrive
├── Standard vs FIFO comparison
└── Terraform & CLI examples
```

---

## Part 1: SQS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW SQS WORKS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Producer → SQS Queue → Consumer                                    │
│                                                                       │
│ ┌───────────┐     ┌──────────┐     ┌───────────┐                  │
│ │ Producer  │────▶│  Queue   │────▶│ Consumer  │                  │
│ │ (sends    │     │ (stores  │     │ (polls &  │                  │
│ │  message) │     │  messages│     │  processes│                  │
│ └───────────┘     │  until   │     │  messages)│                  │
│                   │  consumed│     └───────────┘                  │
│                   └──────────┘                                      │
│                                                                       │
│ Queue types:                                                         │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ Standard      │ FIFO                 │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Throughput           │ Unlimited     │ 300 msg/s (batch:   │   │
│ │                      │               │ 3000 msg/s)          │   │
│ │ Ordering             │ Best-effort   │ Strict FIFO          │   │
│ │ Delivery             │ At-least-once │ Exactly-once         │   │
│ │ Deduplication        │ No            │ Yes (5 min window)   │   │
│ │ Queue name           │ Any           │ Must end in .fifo    │   │
│ │ Use case             │ High throughput│ Order matters       │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
│ Message lifecycle:                                                   │
│ 1. Producer sends message → Message stored in queue              │
│ 2. Consumer polls queue → Receives message                       │
│ 3. Message becomes invisible (visibility timeout starts)         │
│ 4. Consumer processes message                                      │
│ 5. Consumer deletes message → Message removed from queue         │
│ ⚠️ If not deleted within visibility timeout → message reappears │
│                                                                       │
│ Key limits:                                                          │
│ ├── Message size: Max 256 KB                                     │
│ │   → Use SQS Extended Client for larger (stores in S3)        │
│ ├── Retention: 1 minute to 14 days (default 4 days)            │
│ ├── Visibility timeout: 0 seconds to 12 hours (default 30s)   │
│ ├── Long polling: Up to 20 seconds                              │
│ ├── Batch: Up to 10 messages per request                        │
│ └── In-flight messages: 120,000 (standard), 20,000 (FIFO)     │
│                                                                       │
│ Pricing:                                                             │
│ ├── First 1 million requests/month: FREE                        │
│ ├── Standard: $0.40/million requests after free tier           │
│ ├── FIFO: $0.50/million requests                                │
│ └── Data transfer: Standard AWS rates                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Queue (Full Portal Walkthrough)

```
Console → SQS → Create queue

┌─────────────────────────────────────────────────────────────────┐
│           CREATE QUEUE - DETAILS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Type:                                                           │
│ ● Standard                                                     │
│   → Best-effort ordering, at-least-once delivery             │
│   → Unlimited throughput                                      │
│ ○ FIFO                                                         │
│   → Strict ordering, exactly-once delivery                   │
│   → Queue name must end in .fifo                             │
│                                                                   │
│ Name: [order-processing-queue]                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           CONFIGURATION                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Visibility timeout: [30] seconds                               │
│ → Time message is hidden after being received                │
│ → Must be longer than your processing time                   │
│ → If consumer crashes, message reappears after this time     │
│ → Range: 0 seconds to 12 hours                               │
│ → ⚡ Set to 6× your average processing time                  │
│                                                                   │
│ Message retention period: [4] days                             │
│ → How long unprocessed messages stay in queue                │
│ → Range: 1 minute to 14 days                                 │
│ → ⚡ Default 4 days usually sufficient                       │
│                                                                   │
│ Delivery delay: [0] seconds                                    │
│ → Delay before message becomes visible                       │
│ → Range: 0 seconds to 15 minutes                             │
│ → Use case: Delayed processing (wait for related events)    │
│                                                                   │
│ Maximum message size: [256] KB                                 │
│ → Range: 1 KB to 256 KB                                      │
│ → For larger: Use SQS Extended Client Library (stores in S3)│
│                                                                   │
│ Receive message wait time: [20] seconds                       │
│ → 0 = Short polling (returns immediately, may be empty)     │
│ → 1-20 = Long polling (⚡ recommended, reduces costs)       │
│ → ⚡ Always use long polling (20 seconds)                    │
│ → Reduces empty responses and API calls                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           ENCRYPTION                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Server-side encryption:                                        │
│ ● Enabled                                                      │
│ ○ Disabled                                                     │
│                                                                   │
│ Encryption key type:                                           │
│ ● Amazon SQS key (SSE-SQS) (⚡ simplest, free)              │
│ ○ AWS Key Management Service key (SSE-KMS)                   │
│   → Customer managed key: [alias/sqs-key ▼]                │
│   → Data key reuse period: [300] seconds (5 min)            │
│   → ⚠️ KMS calls cost: $0.03/10,000 API calls              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           ACCESS POLICY                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ● Basic (configure who can send/receive)                      │
│                                                                   │
│   Who can send messages to the queue?                         │
│   ● Only the queue owner                                      │
│   ○ Specified AWS accounts, IAM users, and roles             │
│     [Account IDs: 123456789012, 987654321098]               │
│                                                                   │
│   Who can receive messages from the queue?                    │
│   ● Only the queue owner                                      │
│   ○ Specified AWS accounts, IAM users, and roles             │
│                                                                   │
│ ○ Advanced (JSON policy editor)                               │
│   → Fine-grained resource-based policy                       │
│   → Cross-account access                                     │
│   → Condition keys (IP, VPC, encryption)                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           REDRIVE ALLOW POLICY                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Which source queues can use this queue as DLQ?                │
│ ● Allow all                                                    │
│ ○ By queue (specify source queue ARNs)                       │
│ ○ Deny all                                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           DEAD-LETTER QUEUE                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ☑ Enabled                                                      │
│                                                                   │
│ Queue ARN: [arn:aws:sqs:us-east-1:123456:order-dlq ▼]       │
│ → ⚡ Create DLQ first, then reference it here                │
│ → DLQ must be same type (Standard→Standard, FIFO→FIFO)     │
│                                                                   │
│ Maximum receives: [3]                                          │
│ → Number of times a message can be received before DLQ      │
│ → Range: 1 to 1000                                           │
│ → ⚡ Set to 3-5 for most workloads                           │
│ → After N failed processing attempts → moved to DLQ        │
│                                                                   │
│                    [Create queue]                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Message Operations

```
┌─────────────────────────────────────────────────────────────────────┐
│           MESSAGE OPERATIONS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Sending messages (Console → SQS → Queue → Send/receive messages): │
│                                                                       │
│ Message body: [{"orderId": "12345", "item": "widget"}]            │
│ → ⚡ JSON format recommended                                       │
│                                                                       │
│ Message attributes (optional metadata):                             │
│ Name: [OrderType]   Type: [String]   Value: [priority]            │
│ → Up to 10 attributes per message                                 │
│ → Types: String, Number, Binary                                   │
│ → Used for filtering (with SNS subscriptions)                    │
│                                                                       │
│ Message group ID (FIFO only): [order-12345]                        │
│ → Messages in same group processed in order                      │
│                                                                       │
│ Deduplication ID (FIFO only): [unique-id-abc]                     │
│ → Prevents duplicate messages in 5-min window                    │
│                                                                       │
│ Delay delivery: [0] seconds                                         │
│ → Per-message delay (overrides queue default)                    │
│                                                                       │
│ Polling types:                                                       │
│ ├── Short polling (WaitTimeSeconds=0):                           │
│ │   → Returns immediately, may return 0 messages                │
│ │   → Queries subset of SQS servers                             │
│ │   → Higher cost (more empty responses)                        │
│ └── Long polling (WaitTimeSeconds=1-20):                         │
│     → Waits up to N seconds for messages                        │
│     → Queries ALL SQS servers                                    │
│     → ⚡ Lower cost, fewer empty responses                      │
│     → ⚡ Always use long polling                                 │
│                                                                       │
│ Visibility timeout flow:                                             │
│ ┌──────────┐  receive  ┌──────────┐  timeout  ┌──────────┐      │
│ │ Visible  │──────────▶│ Invisible│──────────▶│ Visible  │      │
│ │ (in queue│           │ (being   │  (not     │ (back in │      │
│ │  ready)  │           │ processed│  deleted) │  queue)  │      │
│ └──────────┘           │  by      │           └──────────┘      │
│                        │ consumer)│                                │
│                        └────┬─────┘                                │
│                             │ delete                               │
│                             ▼                                       │
│                        ┌──────────┐                                │
│                        │ Deleted  │                                │
│                        │ (done!)  │                                │
│                        └──────────┘                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Dead-Letter Queues & Redrive

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEAD-LETTER QUEUES (DLQ)                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Separate queue for messages that fail processing              │
│                                                                       │
│ Main Queue ──(max receives exceeded)──▶ Dead-Letter Queue          │
│                                                                       │
│ Why use DLQ:                                                         │
│ ├── Isolate problem messages (don't block good messages)        │
│ ├── Debug failed messages (inspect message content)             │
│ ├── Alert on DLQ depth (CloudWatch alarm)                       │
│ └── Reprocess later (redrive from DLQ back to main queue)      │
│                                                                       │
│ Configuration:                                                       │
│ ├── maxReceiveCount: 3 → after 3 failed attempts, move to DLQ │
│ ├── DLQ must be same type as source queue                       │
│ ├── DLQ retention should be longer (14 days)                   │
│ └── Set CloudWatch alarm on ApproximateNumberOfMessagesVisible │
│                                                                       │
│ DLQ Redrive (Console → SQS → DLQ → Start DLQ redrive):          │
│                                                                       │
│ Redrive destination:                                                │
│ ● Move to source queue(s) (original queue)                       │
│ ○ Move to custom destination                                      │
│                                                                       │
│ Velocity: [System optimized ▼]                                     │
│ → Max number of messages per second to redrive                   │
│ → ⚡ Use "System optimized" for fastest redrive                  │
│                                                                       │
│ [Start redrive] → Messages move back to original queue            │
│                                                                       │
│ ⚡ Always set up DLQ for production queues                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: FIFO Queues

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIFO QUEUE SPECIFICS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Queue name: order-processing.fifo (MUST end in .fifo)              │
│                                                                       │
│ FIFO-specific settings:                                              │
│                                                                       │
│ Content-based deduplication: ☑ Enabled                              │
│ → Uses SHA-256 hash of message body for dedup                    │
│ → No need to provide deduplication ID                            │
│ → ⚡ Enable if message body is unique per message                │
│                                                                       │
│ If disabled: Must provide DeduplicationId per message              │
│                                                                       │
│ High throughput FIFO: ☑ Enabled                                     │
│ → Up to 3,000 messages/second per API action per queue           │
│ → Up to 30,000 messages/second with batching                    │
│ → Requires: FIFO throughput = "High"                             │
│ → Deduplication scope: Message group (not queue)                │
│                                                                       │
│ Message Group ID:                                                    │
│ ├── Required for FIFO queues                                     │
│ ├── Messages in same group processed in strict order            │
│ ├── Different groups processed in parallel                       │
│ ├── Example: Use customer-id as group → orders per customer   │
│ │   are ordered, different customers processed in parallel     │
│ └── ⚡ Use many group IDs for better parallelism                │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────┐       │
│ │ Group A: msg1 → msg2 → msg3  (processed in order)     │       │
│ │ Group B: msg4 → msg5         (processed in order)      │       │
│ │ Group C: msg6 → msg7 → msg8  (processed in order)     │       │
│ │                                                          │       │
│ │ Groups A, B, C processed in parallel!                  │       │
│ └─────────────────────────────────────────────────────────┘       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5.5: SQS + Lambda Integration (Most Common Pattern)

The most common way to process SQS messages is with a **Lambda event source mapping**. Lambda polls the queue, receives batches of messages, and auto-deletes them on success.

```
┌─────────────┐    polls    ┌─────────────┐    auto-deletes    ┌─────────┐
│  SQS Queue  │ ──────────→ │   Lambda    │ ────────────────→ │ Success │
│             │  (batches)  │  Function   │                    └─────────┘
│             │             │             │        ↓ failure
│             │  ←───────── │             │    message returns
│             │  (returns   │             │    to queue (retried)
└─────────────┘   on fail)  └─────────────┘
```

### Console Setup

```
Console → Lambda → Function → Configuration → Triggers → Add trigger
  ┌─────────────────────────────────────────────────────────┐
  │ Trigger configuration                                    │
  │                                                          │
  │ Source:                SQS                               │
  │ SQS queue:            my-queue                          │
  │ Batch size:           10        ← messages per invoke   │
  │ Batch window:         0 sec     ← wait for full batch   │
  │ Report batch item     ☑ Enable  ← IMPORTANT! (see below)│
  │ failures:                                                │
  └─────────────────────────────────────────────────────────┘
```

### Handling Partial Batch Failures

Without `ReportBatchItemFailures`, if 1 out of 10 messages fails, **all 10 are retried**. With it enabled, only the failed message retries.

```python
# Lambda handler with partial batch failure reporting
def handler(event, context):
    batch_item_failures = []
    
    for record in event['Records']:
        try:
            # Process each message
            body = json.loads(record['body'])
            process_order(body)
        except Exception as e:
            # Report this specific message as failed
            batch_item_failures.append({
                "itemIdentifier": record['messageId']
            })
    
    return {"batchItemFailures": batch_item_failures}
```

### Terraform Example

```hcl
resource "aws_lambda_event_source_mapping" "sqs_trigger" {
  event_source_arn                   = aws_sqs_queue.orders.arn
  function_name                      = aws_lambda_function.processor.arn
  batch_size                         = 10
  maximum_batching_window_in_seconds = 5
  
  function_response_types = ["ReportBatchItemFailures"]
}
```

> ⚡ **Key settings:**
> - **Batch size**: 1-10 for FIFO, 1-10,000 for Standard
> - **Batch window**: Wait up to 5 min to fill a batch (reduces invocations)
> - **Concurrency**: Lambda scales up to 1,000 concurrent executions per minute (standard queue)

---

## Part 6: Terraform & CLI Examples

```hcl
# Standard queue with DLQ
resource "aws_sqs_queue" "dlq" {
  name                      = "order-processing-dlq"
  message_retention_seconds = 1209600  # 14 days
}

resource "aws_sqs_queue" "main" {
  name                       = "order-processing"
  visibility_timeout_seconds = 300    # 5 minutes
  message_retention_seconds  = 345600 # 4 days
  receive_wait_time_seconds  = 20     # Long polling
  delay_seconds              = 0
  max_message_size           = 262144 # 256 KB
  sqs_managed_sse_enabled    = true

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3
  })
}

# FIFO queue
resource "aws_sqs_queue" "fifo" {
  name                        = "order-processing.fifo"
  fifo_queue                  = true
  content_based_deduplication = true
  deduplication_scope         = "messageGroup"
  fifo_throughput_limit       = "perMessageGroupId"
  sqs_managed_sse_enabled     = true
}

# Queue policy (allow SNS to send)
resource "aws_sqs_queue_policy" "sns_to_sqs" {
  queue_url = aws_sqs_queue.main.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.main.arn
      Condition = {
        ArnEquals = { "aws:SourceArn" = aws_sns_topic.orders.arn }
      }
    }]
  })
}
```

```bash
# Create queue
aws sqs create-queue --queue-name order-processing \
  --attributes VisibilityTimeout=300,ReceiveMessageWaitTimeSeconds=20

# Send message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/order-processing \
  --message-body '{"orderId":"12345"}' \
  --message-attributes '{"OrderType":{"DataType":"String","StringValue":"priority"}}'

# Receive messages (long polling)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/order-processing \
  --max-number-of-messages 10 \
  --wait-time-seconds 20

# Delete message
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/order-processing \
  --receipt-handle "AQEBwJnK..."

# Get queue attributes
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/order-processing \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Worker Queue (most common)                                │
│ API → SQS → Lambda/EC2 workers → Database                         │
│ ├── API returns 202 Accepted immediately                         │
│ ├── Worker processes async (order processing, email, etc.)      │
│ ├── DLQ for failed messages                                       │
│ └── CloudWatch alarm on DLQ depth                                │
│                                                                       │
│ Pattern 2: Fan-out (SNS + SQS)                                     │
│ SNS Topic → SQS Queue 1 (email service)                           │
│           → SQS Queue 2 (analytics service)                       │
│           → SQS Queue 3 (audit service)                            │
│ ├── One event, multiple consumers                                │
│ ├── Each consumer gets its own copy                              │
│ └── Consumers process independently at own pace                 │
│                                                                       │
│ Pattern 3: Request Buffering                                        │
│ ├── Absorb traffic spikes (queue acts as buffer)                │
│ ├── Producers: Thousands of requests/second                     │
│ ├── Consumers: Process at sustainable rate                       │
│ └── ⚡ Prevents downstream overload                              │
│                                                                       │
│ Pattern 4: Lambda Event Source Mapping                              │
│ SQS → Lambda (auto-triggered)                                      │
│ ├── Lambda polls SQS automatically                               │
│ ├── Batch size: 1-10 messages per invocation                    │
│ ├── Concurrency scales with queue depth                         │
│ └── Failed batch → retry → DLQ                                 │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Always use long polling (20 seconds)                           │
│ 2. Always set up DLQ                                               │
│ 3. Set visibility timeout > processing time                      │
│ 4. Delete messages after successful processing                   │
│ 5. Use batch operations (up to 10) to reduce costs              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common SQS Issues

### "Messages keep reappearing after processing"

This is a **visibility timeout** issue. When a consumer reads a message, the message becomes invisible for `VisibilityTimeout` seconds. If the consumer doesn't delete the message before the timeout expires, it reappears.

```
Solutions:
1. Increase VisibilityTimeout (default 30s) to > your processing time
   Console → SQS → Queue → Edit → Visibility timeout

2. Make sure your code explicitly deletes the message after processing:
   sqs.delete_message(QueueUrl=url, ReceiptHandle=receipt)

3. For Lambda triggers: Ensure your function completes within the timeout
   Lambda auto-deletes on success, returns to queue on failure
```

### "Messages aren't going to the DLQ"

```
1. ☐ Is the DLQ the SAME type as the source queue?
   Standard queue → Standard DLQ (NOT FIFO)
   FIFO queue → FIFO DLQ

2. ☐ Is maxReceiveCount set correctly?
   Console → Queue → Dead-letter queue → Maximum receives
   (e.g., 3 = message moves to DLQ after 3 failed processing attempts)

3. ☐ Does the DLQ exist and have the correct permissions?
```

### "Long polling isn't working (still getting empty responses)"

```
1. ☐ Is WaitTimeSeconds > 0?
   Console → Queue → Edit → Receive message wait time
   Set to 20 seconds (maximum) for best cost efficiency

2. ☐ Is your SDK/code overriding with WaitTimeSeconds=0?
   Check the ReceiveMessage API call parameters
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Short visibility timeout | Messages processed twice | Set timeout > 6x your processing time |
| Not using DLQ | Poison messages retry forever | Always configure a DLQ |
| Short polling (default) | Paying for empty responses | Enable long polling (20s) |
| FIFO queue without .fifo suffix | Queue creation fails | Name must end in `.fifo` |
| Not handling partial batch failures | Entire batch retried for one failure | Use `ReportBatchItemFailures` with Lambda |

---

## Quick Reference

```
SQS Quick Reference:
├── Standard: Unlimited throughput, at-least-once, best-effort order
├── FIFO: 300-3000 msg/s, exactly-once, strict order
├── Message size: Max 256 KB (Extended Client for larger)
├── Retention: 1 min to 14 days (default 4 days)
├── Visibility timeout: 0s to 12 hours (default 30s)
├── Long polling: 1-20 seconds (⚡ always use 20)
├── Batch: Up to 10 messages per send/receive/delete
├── DLQ: Separate queue for failed messages (maxReceiveCount)
├── Encryption: SSE-SQS (free) or SSE-KMS
├── FIFO: Queue name must end in .fifo
├── ⚡ Long polling reduces cost (fewer empty responses)
├── ⚡ Always set up DLQ + CloudWatch alarm
└── ⚡ Fan-out: SNS → multiple SQS queues
```

---

## What's Next?

In **Chapter 46: SNS (Simple Notification Service)**, we'll cover pub/sub messaging, topics, subscriptions, fan-out patterns, and message filtering.
