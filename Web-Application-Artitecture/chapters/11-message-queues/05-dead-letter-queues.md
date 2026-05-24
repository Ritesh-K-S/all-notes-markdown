# Dead Letter Queues — Handling Failed Messages

> **What you'll learn**: What Dead Letter Queues (DLQs) are, why they're essential in any production messaging system, how to configure them in RabbitMQ, Kafka, and SQS, and strategies for handling poison messages that would otherwise crash your consumers forever.

---

## Real-Life Analogy

Imagine a **postal sorting facility**.

Letters come in, and machines try to read the address and route them. But some letters are problematic:
- The address is smudged and unreadable
- The package is too large for the sorting machine
- The destination doesn't exist

What does the post office do? It doesn't throw the letter away. It doesn't try to sort it forever in an infinite loop. Instead, it puts the letter in a special bin called **"Undeliverable Mail" (Dead Letter Office)**.

Later, a human reviews these problematic letters and decides what to do:
- Fix the address and resend
- Return to sender
- Archive for investigation

A **Dead Letter Queue** does exactly this for messages in a software system. When a message can't be processed after several attempts, it's moved to a special queue where humans (or automated systems) can investigate and handle it.

---

## Core Concept Explained Step-by-Step

### Step 1: Why Do Messages Fail?

Messages can fail for many reasons:

```
┌─────────────────────────────────────────────────────────────────┐
│                  REASONS MESSAGES FAIL                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. POISON MESSAGE (bad data)                                    │
│     → Malformed JSON, missing required fields                   │
│     → Will NEVER succeed no matter how many retries             │
│                                                                  │
│  2. TRANSIENT FAILURE                                            │
│     → Database temporarily down, network timeout                │
│     → MIGHT succeed on retry                                    │
│                                                                  │
│  3. BUSINESS LOGIC REJECTION                                     │
│     → Invalid state (e.g., cancel an already-shipped order)     │
│     → Won't succeed, but data is valid                          │
│                                                                  │
│  4. CONSUMER BUG                                                 │
│     → NullPointerException, unhandled edge case                 │
│     → Requires code fix + reprocessing                          │
│                                                                  │
│  5. DEPENDENCY PERMANENTLY DOWN                                  │
│     → Third-party API shut down, endpoint removed               │
│     → Requires architectural change                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: What Happens WITHOUT a DLQ?

```
WITHOUT DEAD LETTER QUEUE (Infinite Loop of Death):

┌──────────┐     ┌──────────┐     ┌──────────┐
│  Queue   │────▶│ Consumer │────▶│ CRASH!   │
│          │     │          │     │          │
│ [poison] │◀────│  (NACK)  │◀────│ Exception│
│ [poison] │     │          │     │          │
│ [poison] │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘
     │                │
     └────────────────┘  ← Infinite retry loop!
     
RESULT:
- Consumer crashes, restarts, crashes again (crash loop)
- Poison message BLOCKS all messages behind it
- Queue depth grows indefinitely
- Legitimate messages are never processed
- You get paged at 3 AM
```

### Step 3: What Happens WITH a DLQ?

```
WITH DEAD LETTER QUEUE (Graceful Failure):

                         Attempt 1: FAIL
                         Attempt 2: FAIL
                         Attempt 3: FAIL ← Max retries reached!
                              │
┌──────────┐     ┌──────────┐│     ┌─────────────────┐
│  Main    │────▶│ Consumer ││────▶│  DEAD LETTER    │
│  Queue   │     │          ││     │     QUEUE       │
│          │     │          │▼     │                 │
│ [msg2]   │     │ Process  │      │ [poison_msg]    │──── Alert team!
│ [msg3]   │     │ msg2 ✓   │      │                 │     Investigate
│ [msg4]   │     │ msg3 ✓   │      │                 │     Fix & replay
└──────────┘     └──────────┘      └─────────────────┘

RESULT:
- Poison message is isolated (moved to DLQ)
- Main queue continues processing normally
- No blockage, no infinite loops
- Team is alerted and can investigate at leisure
- Fixed messages can be replayed from DLQ
```

### Step 4: The Retry + DLQ Flow

A typical production setup uses **exponential backoff retries** before DLQ:

```
Message arrives
       │
       ▼
┌─────────────┐    Success    ┌──────────────┐
│   Attempt 1 │──────────────▶│   DONE ✓     │
│   (0s delay)│               └──────────────┘
└──────┬──────┘
       │ Failure
       ▼
┌─────────────┐    Success    ┌──────────────┐
│   Attempt 2 │──────────────▶│   DONE ✓     │
│   (1s delay)│               └──────────────┘
└──────┬──────┘
       │ Failure
       ▼
┌─────────────┐    Success    ┌──────────────┐
│   Attempt 3 │──────────────▶│   DONE ✓     │
│   (5s delay)│               └──────────────┘
└──────┬──────┘
       │ Failure
       ▼
┌─────────────┐    Success    ┌──────────────┐
│   Attempt 4 │──────────────▶│   DONE ✓     │
│  (30s delay)│               └──────────────┘
└──────┬──────┘
       │ Failure (MAX RETRIES EXHAUSTED)
       ▼
┌─────────────────────────────────────┐
│         DEAD LETTER QUEUE           │
│                                     │
│  Original message                   │
│  + Error reason                     │
│  + Number of attempts               │
│  + Timestamp of each failure        │
│  + Stack trace                      │
└─────────────────────────────────────┘
       │
       ▼
  Alert / Dashboard / Manual review
```

---

## How It Works Internally

### RabbitMQ Dead Letter Exchange (DLX)

RabbitMQ uses a concept called **Dead Letter Exchange** — when a message is dead-lettered, it's published to a special exchange that routes it to the DLQ.

```
┌───────────────────────────────────────────────────────────────────────┐
│                        RABBITMQ DLX FLOW                               │
│                                                                        │
│  ┌──────────┐     ┌───────────┐     ┌──────────────┐                 │
│  │ Producer │────▶│  Main     │────▶│   Consumer   │                 │
│  │          │     │  Exchange │     │              │                 │
│  └──────────┘     └───────────┘     └──────┬───────┘                 │
│                        │                    │                          │
│                        ▼                    │ Message rejected/expired │
│                   ┌──────────┐              │ or queue limit reached  │
│                   │  Main    │              │                          │
│                   │  Queue   │◀─────────────┘                         │
│                   │ (x-dead- │                                        │
│                   │  letter- │──── Dead-letter conditions: ───┐       │
│                   │  exchange│     • basic.reject/nack         │       │
│                   │  = "dlx")│     • TTL expired              │       │
│                   └──────────┘     • Queue max-length reached │       │
│                                                               ▼       │
│                                    ┌───────────────────────────┐      │
│                                    │  Dead Letter Exchange     │      │
│                                    │  (DLX)                    │      │
│                                    └─────────────┬─────────────┘      │
│                                                  │                    │
│                                                  ▼                    │
│                                    ┌───────────────────────────┐      │
│                                    │  Dead Letter Queue        │      │
│                                    │  [failed_msg_1]           │      │
│                                    │  [failed_msg_2]           │      │
│                                    └───────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────┘

Messages are dead-lettered when:
1. Consumer rejects (basic.reject or basic.nack) with requeue=false
2. Message TTL expires
3. Queue reaches max-length limit
```

### Amazon SQS Redrive Policy

SQS uses a **Redrive Policy** that counts how many times a message has been received:

```
┌───────────────────────────────────────────────────────────────┐
│                    SQS DLQ MECHANISM                            │
│                                                                │
│  Main Queue (visibility timeout = 60s)                        │
│  Redrive Policy: maxReceiveCount = 3                          │
│                                                                │
│  ┌──────────────────────────────────────────┐                 │
│  │ Message "ORD-123"                         │                 │
│  │ receiveCount: 0 → 1 → 2 → 3             │                 │
│  │                                           │                 │
│  │ Receive #1: Consumer reads, fails,        │                 │
│  │            doesn't delete → visibility    │                 │
│  │            timeout expires → visible again │                 │
│  │                                           │                 │
│  │ Receive #2: Another consumer reads,       │                 │
│  │            fails again → visible again    │                 │
│  │                                           │                 │
│  │ Receive #3: receiveCount = 3 =            │                 │
│  │            maxReceiveCount!               │                 │
│  │            → MOVED TO DLQ automatically   │                 │
│  └──────────────────────────────────────────┘                 │
│                         │                                      │
│                         ▼                                      │
│  ┌──────────────────────────────────────────┐                 │
│  │  Dead Letter Queue                        │                 │
│  │  Message "ORD-123" + metadata            │                 │
│  │  (original queue, timestamps, etc.)       │                 │
│  └──────────────────────────────────────────┘                 │
└───────────────────────────────────────────────────────────────┘
```

### Kafka — DLQ Pattern (Manual Implementation)

Kafka doesn't have built-in DLQ support, but the pattern is implemented manually:

```
┌──────────────────────────────────────────────────────────────────┐
│                KAFKA DLQ PATTERN                                   │
│                                                                    │
│  Topic: "orders"                                                  │
│  ┌───┬───┬───┬───┬───┐                                          │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │                                          │
│  └───┴───┴───┴───┴───┘                                          │
│           │                                                       │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │    Consumer     │                                             │
│  │                 │── Try processing ──┐                        │
│  │                 │                    │                         │
│  │  if (retries    │◀── Failed! ────────┘                        │
│  │    < MAX) {     │                                             │
│  │    retry with   │── Publish to "orders.retry" topic          │
│  │    delay        │                                             │
│  │  } else {       │                                             │
│  │    send to DLQ  │── Publish to "orders.dlq" topic            │
│  │  }              │                                             │
│  └─────────────────┘                                             │
│                                                                    │
│  Topic: "orders.retry"   Topic: "orders.dlq"                     │
│  ┌───┬───┬───┐           ┌───┬───┐                               │
│  │ 0 │ 1 │ 2 │           │ 0 │ 1 │  ← Permanently failed       │
│  └───┴───┴───┘           └───┴───┘    messages stored here       │
│       │                                                           │
│       │ (delayed reprocessing)                                    │
│       ▼                                                           │
│  Back to consumer for retry                                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — RabbitMQ with DLQ

```python
import pika
import json
import traceback

# ============ SETUP: Create queues with DLQ ============
def setup_queues_with_dlq():
    """Set up main queue with Dead Letter Exchange"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()
    
    # 1. Declare Dead Letter Exchange and Queue
    channel.exchange_declare(exchange='dlx', exchange_type='direct')
    channel.queue_declare(queue='orders_dlq', durable=True)
    channel.queue_bind(queue='orders_dlq', exchange='dlx', routing_key='orders')
    
    # 2. Declare Main Queue with DLQ configuration
    channel.queue_declare(
        queue='orders',
        durable=True,
        arguments={
            'x-dead-letter-exchange': 'dlx',         # Where to send dead letters
            'x-dead-letter-routing-key': 'orders',   # Routing key for DLQ
            'x-message-ttl': 60000,                  # Optional: TTL (60s)
        }
    )
    print("Queues configured with DLQ support!")
    connection.close()


# ============ CONSUMER with retry logic ============
MAX_RETRIES = 3

def consume_with_retry():
    """Consumer that retries and dead-letters on failure"""
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()
    channel.basic_qos(prefetch_count=1)
    
    def callback(ch, method, properties, body):
        # Track retry count via message headers
        headers = properties.headers or {}
        retry_count = headers.get('x-retry-count', 0)
        
        try:
            order = json.loads(body)
            process_order(order)  # Business logic
            ch.basic_ack(delivery_tag=method.delivery_tag)
            print(f"✓ Processed order: {order['order_id']}")
            
        except Exception as e:
            retry_count += 1
            
            if retry_count >= MAX_RETRIES:
                # Max retries reached → reject (goes to DLQ via DLX)
                print(f"✗ DEAD LETTERED after {retry_count} attempts: {e}")
                ch.basic_reject(delivery_tag=method.delivery_tag, requeue=False)
            else:
                # Retry: republish with incremented counter
                print(f"⟳ Retry {retry_count}/{MAX_RETRIES}: {e}")
                ch.basic_publish(
                    exchange='',
                    routing_key='orders',
                    body=body,
                    properties=pika.BasicProperties(
                        headers={'x-retry-count': retry_count,
                                 'x-last-error': str(e)},
                        delivery_mode=2
                    )
                )
                ch.basic_ack(delivery_tag=method.delivery_tag)
    
    channel.basic_consume(queue='orders', on_message_callback=callback)
    channel.start_consuming()


def process_order(order):
    """Simulated processing that might fail"""
    if 'amount' not in order:
        raise ValueError("Missing 'amount' field — poison message!")
    # ... actual processing
```

### Python — SQS with DLQ (Boto3)

```python
import boto3
import json
import time

# ============ SETUP: Create SQS queue with DLQ ============
def create_queue_with_dlq():
    """Create an SQS queue with a Dead Letter Queue"""
    sqs = boto3.client('sqs', region_name='us-east-1')
    
    # Create DLQ first
    dlq_response = sqs.create_queue(
        QueueName='orders-dlq',
        Attributes={'MessageRetentionPeriod': '1209600'}  # 14 days
    )
    dlq_url = dlq_response['QueueUrl']
    
    # Get DLQ ARN
    dlq_attrs = sqs.get_queue_attributes(
        QueueUrl=dlq_url, AttributeNames=['QueueArn']
    )
    dlq_arn = dlq_attrs['Attributes']['QueueArn']
    
    # Create main queue with Redrive Policy
    main_response = sqs.create_queue(
        QueueName='orders',
        Attributes={
            'VisibilityTimeout': '60',
            'RedrivePolicy': json.dumps({
                'deadLetterTargetArn': dlq_arn,
                'maxReceiveCount': '3'  # After 3 failed attempts → DLQ
            })
        }
    )
    return main_response['QueueUrl'], dlq_url


# ============ DLQ MONITOR: Process failed messages ============
def monitor_dlq(dlq_url):
    """Monitor DLQ and alert on new dead-lettered messages"""
    sqs = boto3.client('sqs', region_name='us-east-1')
    
    while True:
        response = sqs.receive_message(
            QueueUrl=dlq_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,
            MessageAttributeNames=['All'],
            AttributeNames=['All']  # Get metadata (receive count, timestamps)
        )
        
        for msg in response.get('Messages', []):
            body = json.loads(msg['Body'])
            receive_count = msg['Attributes'].get('ApproximateReceiveCount', '?')
            
            print(f"⚠️  DEAD LETTER: {body}")
            print(f"   Receive count: {receive_count}")
            print(f"   MessageId: {msg['MessageId']}")
            
            # Options: Alert team, log to monitoring, attempt manual fix
            alert_ops_team(body, msg['MessageId'])
            
            # After handling, delete from DLQ
            sqs.delete_message(
                QueueUrl=dlq_url,
                ReceiptHandle=msg['ReceiptHandle']
            )
```

### Java — Kafka DLQ Pattern

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.producer.*;
import java.time.Duration;
import java.util.Collections;

public class KafkaConsumerWithDLQ {
    
    private static final int MAX_RETRIES = 3;
    private static final String MAIN_TOPIC = "orders";
    private static final String DLQ_TOPIC = "orders.dlq";
    
    private final KafkaConsumer<String, String> consumer;
    private final KafkaProducer<String, String> dlqProducer;
    
    public void processWithDLQ() {
        consumer.subscribe(Collections.singletonList(MAIN_TOPIC));
        
        while (true) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(1000));
            
            for (ConsumerRecord<String, String> record : records) {
                boolean success = processWithRetry(record);
                
                if (!success) {
                    // All retries exhausted → send to DLQ
                    sendToDLQ(record);
                }
            }
            consumer.commitSync();
        }
    }
    
    private boolean processWithRetry(ConsumerRecord<String, String> record) {
        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                processOrder(record.value());
                return true;  // Success!
            } catch (Exception e) {
                System.err.printf("Attempt %d/%d failed for key=%s: %s%n",
                    attempt, MAX_RETRIES, record.key(), e.getMessage());
                
                if (attempt < MAX_RETRIES) {
                    // Exponential backoff: 1s, 4s, 9s
                    sleep(attempt * attempt * 1000L);
                }
            }
        }
        return false;  // All retries failed
    }
    
    private void sendToDLQ(ConsumerRecord<String, String> record) {
        ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
            DLQ_TOPIC, record.key(), record.value()
        );
        // Add metadata headers
        dlqRecord.headers()
            .add("original-topic", MAIN_TOPIC.getBytes())
            .add("original-partition", 
                 String.valueOf(record.partition()).getBytes())
            .add("original-offset", 
                 String.valueOf(record.offset()).getBytes())
            .add("failure-timestamp", 
                 String.valueOf(System.currentTimeMillis()).getBytes());
        
        dlqProducer.send(dlqRecord);
        System.out.println("☠ Sent to DLQ: key=" + record.key());
    }
}
```

### Java — Spring Kafka with DLQ (Declarative)

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.RetryableTopic;
import org.springframework.kafka.retrytopic.TopicSuffixingStrategy;
import org.springframework.retry.annotation.Backoff;
import org.springframework.stereotype.Service;

@Service
public class OrderConsumerWithRetry {
    
    // Spring Kafka automatically creates retry topics and DLQ!
    @RetryableTopic(
        attempts = "4",                              // 1 original + 3 retries
        backoff = @Backoff(delay = 1000, multiplier = 2),  // 1s, 2s, 4s
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE
    )
    @KafkaListener(topics = "orders", groupId = "order-processing")
    public void processOrder(String orderJson) {
        Order order = parseOrder(orderJson);
        
        if (order.getAmount() == null) {
            throw new RuntimeException("Invalid order: missing amount");
            // Spring will retry 3 times, then send to "orders-dlt" (DLQ)
        }
        
        // Process normally
        paymentService.charge(order);
    }
    
    // Handler for messages that end up in DLQ
    @KafkaListener(topics = "orders-dlt", groupId = "dlq-handler")
    public void handleDeadLetter(String orderJson) {
        System.out.println("⚠️ DLQ Message received: " + orderJson);
        // Log to monitoring system, alert team, store for review
        monitoringService.alertDeadLetter("orders", orderJson);
    }
}
```

---

## Infrastructure Examples

### RabbitMQ DLQ with Docker Compose

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3.12-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret
    volumes:
      - ./rabbitmq-definitions.json:/etc/rabbitmq/definitions.json
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
```

```json
// rabbitmq-definitions.json — Pre-configure DLQ on startup
{
  "exchanges": [
    {"name": "dlx", "type": "direct", "durable": true}
  ],
  "queues": [
    {
      "name": "orders",
      "durable": true,
      "arguments": {
        "x-dead-letter-exchange": "dlx",
        "x-dead-letter-routing-key": "orders-dead"
      }
    },
    {
      "name": "orders-dlq",
      "durable": true
    }
  ],
  "bindings": [
    {"source": "dlx", "destination": "orders-dlq", 
     "routing_key": "orders-dead", "destination_type": "queue"}
  ]
}
```

### AWS SQS DLQ with Terraform

```hcl
# Dead Letter Queue
resource "aws_sqs_queue" "orders_dlq" {
  name                      = "orders-dlq"
  message_retention_seconds = 1209600  # 14 days (max)
  
  tags = {
    Purpose = "Dead letter queue for failed order messages"
  }
}

# Main Queue with Redrive Policy
resource "aws_sqs_queue" "orders" {
  name                       = "orders"
  visibility_timeout_seconds = 60
  receive_wait_time_seconds  = 20
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3
  })
}

# CloudWatch Alarm when DLQ has messages (alert the team!)
resource "aws_cloudwatch_metric_alarm" "dlq_alarm" {
  alarm_name          = "orders-dlq-has-messages"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Alert when dead letter queue receives messages"
  
  dimensions = {
    QueueName = aws_sqs_queue.orders_dlq.name
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]  # Notify team
}
```

---

## Real-World Example

### Stripe — Payment Processing DLQ

Stripe processes millions of payment webhooks daily. Their system uses DLQs extensively:

```
STRIPE WEBHOOK DLQ ARCHITECTURE:

Payment event occurs
        │
        ▼
┌───────────────────┐     ┌─────────────────────────────────┐
│  Webhook Queue    │────▶│  Webhook Delivery Service        │
│  [event1]         │     │                                 │
│  [event2]         │     │  Try delivery to merchant URL:  │
│  [event3]         │     │    Attempt 1: timeout           │
└───────────────────┘     │    Attempt 2: 500 error         │
                          │    Attempt 3: timeout           │
                          │    ...                          │
                          │    Attempt 8: (over 72 hours)   │
                          └────────────────┬────────────────┘
                                           │
                                  Max retries exhausted
                                           │
                                           ▼
                          ┌─────────────────────────────────┐
                          │  Dead Letter Queue               │
                          │  + Alert merchant dashboard     │
                          │  + Flag in Stripe dashboard     │
                          │  + Available for manual replay  │
                          └─────────────────────────────────┘

Stripe retries with exponential backoff over 72 hours before 
dead-lettering a webhook.
```

### Netflix — Async Job Processing

Netflix uses DLQs for their content processing pipeline:
- Video transcoding jobs that fail (corrupted source, codec error)
- Metadata extraction failures
- Recommendation model updates that crash

Failed jobs go to DLQ → engineering team investigates → fixes are applied → messages are replayed from DLQ.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| No DLQ configured at all | Poison messages cause infinite retry loops | ALWAYS configure a DLQ for production queues |
| Not monitoring DLQ depth | Messages pile up silently, nobody notices | Set up alerts when DLQ has > 0 messages |
| DLQ with no investigation process | Dead letters pile up, eventually expire | Create runbooks for DLQ investigation |
| maxReceiveCount = 1 | Messages dead-lettered on first transient failure | Set 3-5 retries with exponential backoff |
| DLQ retention too short | Messages expire before team can investigate | Set 14-day retention on DLQs |
| No metadata in DLQ messages | Hard to debug why message failed | Include: error message, stack trace, attempt count, timestamps |
| Replaying DLQ without fixing root cause | Messages fail again immediately | Fix the bug/issue FIRST, then replay |
| Same consumer group for DLQ and main queue | DLQ messages compete with main traffic | Separate consumer group for DLQ processing |

---

## When to Use / When NOT to Use

### ✅ Use Dead Letter Queues When:

- **Any production message queue** — DLQ is a safety net, always configure one
- **Payment processing** — can't lose financial transactions
- **Order processing** — must handle every order eventually
- **Webhook delivery** — retry with backoff, dead-letter after exhausting retries
- **Async job processing** — jobs that fail need investigation
- **Multi-service architectures** — where transient failures are common

### ❌ When DLQ Might Not Be Needed:

- **Ephemeral messages that are OK to lose** (real-time metrics with TTL)
- **Idempotent operations with auto-recovery** (cache warming)
- **Development/testing environments** (where manual investigation isn't needed)
- **Stream processing with built-in error handling** (Kafka Streams error handlers)

### DLQ Strategy Guide:

| Scenario | maxRetries | Backoff | DLQ Action |
|----------|-----------|---------|------------|
| Payment processing | 5 | Exponential (1s, 5s, 30s, 2min, 10min) | Alert immediately, manual review |
| Email sending | 3 | Linear (10s, 20s, 30s) | Retry next day, then alert |
| Analytics events | 2 | None | Log and discard (non-critical) |
| Inventory updates | 4 | Exponential | Alert, auto-replay after fix |
| Webhook delivery | 8 | Exponential (over 72 hours) | Notify merchant, flag in dashboard |

---

## Key Takeaways

- A **Dead Letter Queue** is a safety net that catches messages that can't be processed after multiple attempts.
- Without a DLQ, **poison messages** cause infinite retry loops that block ALL subsequent messages.
- **RabbitMQ** uses Dead Letter Exchanges (DLX) — rejected/expired messages are routed to a DLQ via exchange bindings.
- **SQS** uses Redrive Policies — after `maxReceiveCount` failures, messages automatically move to DLQ.
- **Kafka** requires manual DLQ implementation — produce failed messages to a separate `.dlq` topic.
- Always include **metadata** in dead-lettered messages: error reason, attempt count, timestamps, original queue.
- **Monitor DLQ depth** and alert immediately when messages arrive — every DLQ message represents a failure.

---

## What's Next?

Dead Letter Queues handle the _failure_ case. But there's a bigger question: when a message IS delivered, how many times? Exactly once? At least once? At most once? In the next chapter, [06-delivery-guarantees.md](./06-delivery-guarantees.md), we'll explore **Delivery Guarantees** — the fundamental trade-offs that determine whether your system can lose, duplicate, or perfectly deliver every message.
