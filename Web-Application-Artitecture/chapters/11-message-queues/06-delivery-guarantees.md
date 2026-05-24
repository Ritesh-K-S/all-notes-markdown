# Exactly-Once, At-Least-Once, At-Most-Once Delivery

> **What you'll learn**: The three fundamental message delivery guarantees in distributed systems, why "exactly-once" is so hard (some say impossible), and the practical strategies used by Kafka, RabbitMQ, and SQS to achieve reliable delivery in production.

---

## Real-Life Analogy

Imagine you're sending an important **wedding invitation** to your friend.

**At-Most-Once** (Send and forget):
You drop the letter in the mailbox and walk away. Maybe it arrives, maybe it gets lost. You don't check or resend. Your friend might never get it.

**At-Least-Once** (Send until confirmed):
You send the letter. If you don't get an RSVP within a week, you send another copy. And another. Your friend might get 3 identical invitations, but they DEFINITELY get at least one.

**Exactly-Once** (Perfect delivery):
You send the letter, and through some magical postal system, your friend gets exactly ONE invitation — guaranteed. No loss, no duplicates.

Now here's the twist: in the real world, **exactly-once delivery of physical mail is impossible**. The postal system can't guarantee a letter isn't lost without sometimes delivering duplicates. The same fundamental challenge exists in distributed computer systems.

---

## Core Concept Explained Step-by-Step

### Step 1: Why Is This Even a Problem?

In a distributed system, network failures happen constantly. When a producer sends a message to a broker:

```
SCENARIO: Producer sends message, gets no ACK back

Producer                    Broker
   │                          │
   │── send(message) ────────▶│
   │                          │── stores message ✓
   │                          │
   │     ✗ NETWORK DIES ✗     │── sends ACK ─── ✗ (lost!)
   │                          │
   │  (never got ACK...)      │
   │                          │
   │  Did the broker get it?  │  YES! But producer doesn't know!
   │  Should I resend?        │
   └──────────────────────────┘

IF producer resends → DUPLICATE (broker already has it)
IF producer doesn't resend → possible LOSS (what if broker didn't get it?)
```

This is the **fundamental dilemma**: without confirmation, you can't distinguish between "message lost" and "ACK lost."

### Step 2: The Three Guarantees

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  AT-MOST-ONCE                                                       │
│  ═══════════                                                        │
│  "Fire and forget"                                                  │
│                                                                      │
│  Producer ──▶ Broker ──▶ Consumer                                   │
│       │                      │                                      │
│  No retry.              No ACK needed.                              │
│  If lost, it's lost.    Process once or not at all.                 │
│                                                                      │
│  Messages delivered: 0 or 1 time                                    │
│  Risk: MESSAGE LOSS                                                 │
│  Benefit: Fastest, simplest                                         │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  AT-LEAST-ONCE                                                      │
│  ══════════════                                                     │
│  "Keep trying until confirmed"                                      │
│                                                                      │
│  Producer ──▶ Broker ──▶ Consumer                                   │
│       │          │           │                                      │
│  Retries if   Redelivers   ACK after                                │
│  no ACK.      if no ACK.   processing.                              │
│                                                                      │
│  Messages delivered: 1 or MORE times                                │
│  Risk: DUPLICATES                                                   │
│  Benefit: No message loss                                           │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  EXACTLY-ONCE                                                       │
│  ═════════════                                                      │
│  "Delivered precisely one time — no loss, no duplicates"            │
│                                                                      │
│  Producer ──▶ Broker ──▶ Consumer                                   │
│       │          │           │                                      │
│  Dedup at    Dedup at    Idempotent                                 │
│  producer.   broker.     processing.                                │
│                                                                      │
│  Messages delivered: EXACTLY 1 time                                 │
│  Risk: COMPLEXITY (very hard to achieve)                            │
│  Benefit: Perfect semantics                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 3: Visual Comparison

```
Sent: [A] [B] [C] [D] [E]

AT-MOST-ONCE delivery:
Received: [A] [_] [C] [_] [E]     ← B and D were LOST (acceptable)

AT-LEAST-ONCE delivery:
Received: [A] [B] [B] [C] [D] [D] [D] [E]     ← B and D DUPLICATED

EXACTLY-ONCE delivery:
Received: [A] [B] [C] [D] [E]     ← Perfect (but HARD to achieve)
```

### Step 4: The CAP Theorem Connection

Why is exactly-once so hard? Because in a distributed system:
- Networks are unreliable (packets get lost)
- Nodes can crash at any moment
- There's no global clock to coordinate state

The **Two Generals Problem** and **FLP Impossibility** theorem prove that perfect consensus in an asynchronous network is fundamentally impossible. "Exactly-once delivery" across network boundaries is, strictly speaking, **impossible**. What we CAN achieve is **exactly-once _processing_** (semantically equivalent) through idempotency.

---

## How It Works Internally

### At-Most-Once Implementation

```
PRODUCER SIDE:
┌──────────────────────────────────────┐
│  send(message)                       │
│  // Don't wait for ACK              │
│  // Don't retry                     │
│  // Move on immediately             │
│                                      │
│  Kafka: acks=0                      │
│  RabbitMQ: No confirms, no persist  │
│  SQS: N/A (always at-least-once)   │
└──────────────────────────────────────┘

CONSUMER SIDE:
┌──────────────────────────────────────┐
│  receive(message)                    │
│  commit_offset()  // BEFORE processing│
│  process(message)  // might fail!    │
│                                      │
│  If process() crashes after commit:  │
│  → message offset already committed  │
│  → message will NOT be redelivered   │
│  → message effectively LOST          │
└──────────────────────────────────────┘
```

**Use case**: Metrics, logs, real-time analytics where losing some data is acceptable for speed.

### At-Least-Once Implementation

```
PRODUCER SIDE:
┌──────────────────────────────────────┐
│  send(message)                       │
│  wait_for_ack()                      │
│  if (timeout or error):             │
│      RETRY send(message)  // might  │
│                           // cause  │
│                           // dups!  │
│                                      │
│  Kafka: acks=all, retries=3         │
│  RabbitMQ: Publisher confirms ON     │
│  SQS: Built-in (always retries)    │
└──────────────────────────────────────┘

CONSUMER SIDE:
┌──────────────────────────────────────┐
│  receive(message)                    │
│  process(message)  // do work first │
│  commit_offset()   // AFTER success │
│                                      │
│  If crash BETWEEN process & commit: │
│  → offset not committed             │
│  → message REDELIVERED on restart   │
│  → processed AGAIN (duplicate!)     │
└──────────────────────────────────────┘
```

```
TIMELINE — Why duplicates happen:

Consumer                           Broker
   │                                 │
   │◀── deliver message M ──────────│
   │                                 │
   │── process(M) ✓ ────────────    │
   │                            │    │
   │   ✗ CRASH! ✗              │    │
   │   (before commit)          │    │
   │                                 │
   │ ... restart ...                 │
   │                                 │
   │◀── redeliver message M ────────│  (broker thinks M wasn't processed)
   │                                 │
   │── process(M) ✓ AGAIN!          │  (DUPLICATE processing!)
   │── commit ─────────────────────▶│
```

**Use case**: Most production systems (payments, orders, notifications) — duplicates are handled via idempotency.

### Exactly-Once Processing (The Holy Grail)

True exactly-once DELIVERY is impossible over unreliable networks. But we can achieve **exactly-once PROCESSING** using:

**Strategy 1: Idempotent Producer (Kafka)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KAFKA IDEMPOTENT PRODUCER                          │
│                                                                      │
│  Producer assigns sequence numbers to each message:                  │
│                                                                      │
│  Producer ──▶ [seq=1, msg=A] ──▶ Broker                            │
│  Producer ──▶ [seq=2, msg=B] ──▶ Broker                            │
│  Producer ──▶ [seq=2, msg=B] ──▶ Broker  ← RETRY (network issue)  │
│                                                                      │
│  Broker sees seq=2 AGAIN → "Already have this!" → DEDUP!           │
│                                                                      │
│  Result: Only one copy of msg B stored. No duplicates.              │
│                                                                      │
│  Config: enable.idempotence=true                                    │
│  Internally: ProducerID + SequenceNumber + Partition → dedup key    │
└─────────────────────────────────────────────────────────────────────┘
```

**Strategy 2: Transactional Producer + Consumer (Kafka)**

```
┌─────────────────────────────────────────────────────────────────────┐
│              KAFKA TRANSACTIONS (Read-Process-Write)                  │
│                                                                      │
│  Consumer reads from Topic A                                        │
│       │                                                              │
│       ▼                                                              │
│  BEGIN TRANSACTION                                                   │
│       │                                                              │
│       ├── Process message                                           │
│       ├── Write result to Topic B                                   │
│       ├── Commit consumer offset for Topic A                        │
│       │                                                              │
│  COMMIT TRANSACTION (atomic — all or nothing!)                      │
│                                                                      │
│  If crash before COMMIT:                                            │
│  → Transaction is ABORTED                                           │
│  → No partial state (neither write nor offset commit persists)      │
│  → On restart: reads same message, processes again                  │
│  → But output is idempotent (same result written to Topic B)        │
│                                                                      │
│  Result: Exactly-once SEMANTICS (from Topic A to Topic B)           │
└─────────────────────────────────────────────────────────────────────┘
```

**Strategy 3: Idempotent Consumer (Application Level)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                  IDEMPOTENT CONSUMER PATTERN                          │
│                                                                      │
│  For EVERY message, check: "Have I processed this before?"          │
│                                                                      │
│  ┌──────────┐    ┌───────────────┐    ┌──────────────────────┐     │
│  │ Message  │───▶│ Idempotency   │───▶│   Already processed? │     │
│  │ arrives  │    │ Key Check     │    │                      │     │
│  └──────────┘    └───────────────┘    │  YES → Skip (dedup) │     │
│                                       │  NO  → Process       │     │
│                                       └──────────────────────┘     │
│                                                                      │
│  Idempotency Key Storage:                                           │
│  ┌────────────────────────────────────────┐                         │
│  │  Redis/DB: processed_messages          │                         │
│  │  ┌──────────────┬─────────────────┐    │                         │
│  │  │ message_id   │ processed_at    │    │                         │
│  │  ├──────────────┼─────────────────┤    │                         │
│  │  │ msg-001      │ 2024-01-15 10:30│    │                         │
│  │  │ msg-002      │ 2024-01-15 10:31│    │                         │
│  │  │ msg-003      │ 2024-01-15 10:32│    │                         │
│  │  └──────────────┴─────────────────┘    │                         │
│  └────────────────────────────────────────┘                         │
│                                                                      │
│  New message arrives with id "msg-002":                             │
│  → Check Redis: EXISTS? YES → SKIP (already processed)             │
│                                                                      │
│  New message arrives with id "msg-004":                             │
│  → Check Redis: EXISTS? NO → PROCESS + store "msg-004"             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — At-Most-Once (Kafka, acks=0)

```python
from confluent_kafka import Producer

# AT-MOST-ONCE: Fire and forget, fastest but may lose messages
def send_metrics_at_most_once(metric_data):
    """For non-critical data where speed matters more than reliability"""
    producer = Producer({
        'bootstrap.servers': 'localhost:9092',
        'acks': '0',          # Don't wait for ANY acknowledgment
        'retries': 0,         # Never retry
        'linger.ms': 0,       # Send immediately
    })
    
    # No callback, no waiting — fastest possible
    producer.produce('metrics', value=json.dumps(metric_data).encode())
    producer.flush()  # Just flush the buffer, don't wait for ack
```

### Python — At-Least-Once (Kafka, acks=all + retries)

```python
from confluent_kafka import Producer, Consumer
import json

# AT-LEAST-ONCE PRODUCER: Retry until confirmed
def send_order_at_least_once(order):
    """For critical data — guaranteed delivery, may have duplicates"""
    producer = Producer({
        'bootstrap.servers': 'localhost:9092',
        'acks': 'all',            # Wait for ALL replicas to confirm
        'retries': 5,             # Retry up to 5 times on failure
        'retry.backoff.ms': 100,  # Wait between retries
    })
    
    def on_delivery(err, msg):
        if err:
            print(f"DELIVERY FAILED: {err} — will be retried automatically")
        else:
            print(f"Delivered to {msg.topic()}[{msg.partition()}]@{msg.offset()}")
    
    producer.produce('orders', value=json.dumps(order).encode(),
                     callback=on_delivery)
    producer.flush()


# AT-LEAST-ONCE CONSUMER: Commit AFTER processing
def consume_orders_at_least_once():
    """Process then commit — safe but may see duplicates"""
    consumer = Consumer({
        'bootstrap.servers': 'localhost:9092',
        'group.id': 'order-processor',
        'enable.auto.commit': False,  # MANUAL commit only!
        'auto.offset.reset': 'earliest',
    })
    consumer.subscribe(['orders'])
    
    while True:
        msg = consumer.poll(1.0)
        if msg is None:
            continue
        
        # Step 1: PROCESS the message
        order = json.loads(msg.value())
        process_order(order)  # If this crashes, message will be redelivered
        
        # Step 2: COMMIT only after successful processing
        consumer.commit(asynchronous=False)
        # If crash between process and commit → DUPLICATE on restart
```

### Python — Exactly-Once Processing (Idempotent Consumer)

```python
import redis
import json
from confluent_kafka import Consumer

# EXACTLY-ONCE PROCESSING via idempotent consumer
class IdempotentOrderProcessor:
    """Ensures each order is processed exactly once using Redis dedup"""
    
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379)
        self.consumer = Consumer({
            'bootstrap.servers': 'localhost:9092',
            'group.id': 'order-processor',
            'enable.auto.commit': False,
        })
        self.consumer.subscribe(['orders'])
    
    def process_messages(self):
        while True:
            msg = self.consumer.poll(1.0)
            if msg is None:
                continue
            
            order = json.loads(msg.value())
            message_id = order['order_id']  # Unique identifier
            
            # IDEMPOTENCY CHECK: Have we processed this before?
            dedup_key = f"processed:{message_id}"
            
            if self.redis.exists(dedup_key):
                # DUPLICATE! Skip processing, just commit offset
                print(f"⏭ Skipping duplicate: {message_id}")
                self.consumer.commit(asynchronous=False)
                continue
            
            # First time seeing this message — process it
            try:
                self._process_order(order)
                
                # Mark as processed (with TTL for cleanup)
                self.redis.setex(dedup_key, 86400 * 7, "1")  # 7 days TTL
                
                # Commit offset
                self.consumer.commit(asynchronous=False)
                print(f"✓ Processed: {message_id}")
                
            except Exception as e:
                # Don't commit — message will be redelivered
                print(f"✗ Failed: {message_id} — {e}")
                # On retry: idempotency check prevents double-processing
    
    def _process_order(self, order):
        """Business logic — charge payment, update inventory, etc."""
        # Use database transaction for atomicity
        with db.transaction():
            db.execute("UPDATE inventory SET qty = qty - ? WHERE item = ?",
                      (order['qty'], order['item']))
            db.execute("INSERT INTO orders (id, status) VALUES (?, 'confirmed')",
                      (order['order_id'],))
```

### Java — Kafka Exactly-Once with Transactions

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import java.util.*;

public class ExactlyOnceProcessor {
    
    private final KafkaProducer<String, String> producer;
    private final KafkaConsumer<String, String> consumer;
    
    public ExactlyOnceProcessor() {
        // PRODUCER with idempotence + transactions
        Properties prodProps = new Properties();
        prodProps.put("bootstrap.servers", "localhost:9092");
        prodProps.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        prodProps.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        prodProps.put("enable.idempotence", "true");        // Dedup at broker
        prodProps.put("transactional.id", "order-txn-1");   // Enable transactions
        prodProps.put("acks", "all");
        
        this.producer = new KafkaProducer<>(prodProps);
        this.producer.initTransactions();  // Initialize transactional producer
        
        // CONSUMER with read-committed isolation
        Properties consProps = new Properties();
        consProps.put("bootstrap.servers", "localhost:9092");
        consProps.put("group.id", "order-processor");
        consProps.put("isolation.level", "read_committed");  // Only read committed msgs
        consProps.put("enable.auto.commit", "false");
        
        this.consumer = new KafkaConsumer<>(consProps);
        this.consumer.subscribe(Collections.singletonList("raw-orders"));
    }
    
    /**
     * Read-Process-Write pattern with exactly-once semantics.
     * Reads from "raw-orders", processes, writes to "processed-orders".
     * The read offset commit + write are ATOMIC (all-or-nothing).
     */
    public void processExactlyOnce() {
        while (true) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(1000));
            
            if (records.isEmpty()) continue;
            
            // BEGIN TRANSACTION
            producer.beginTransaction();
            
            try {
                for (ConsumerRecord<String, String> record : records) {
                    // Process the record
                    String result = processOrder(record.value());
                    
                    // Write result to output topic (within transaction)
                    producer.send(new ProducerRecord<>(
                        "processed-orders", record.key(), result
                    ));
                }
                
                // Commit consumer offsets within the SAME transaction
                producer.sendOffsetsToTransaction(
                    getOffsets(records),           // Current offsets
                    consumer.groupMetadata()       // Consumer group
                );
                
                // COMMIT TRANSACTION (atomic: offsets + produced messages)
                producer.commitTransaction();
                
            } catch (Exception e) {
                // ABORT: Nothing persists (no partial state)
                producer.abortTransaction();
                System.err.println("Transaction aborted: " + e.getMessage());
            }
        }
    }
}
```

### Java — Idempotent Consumer with Database

```java
import org.springframework.transaction.annotation.Transactional;
import org.springframework.kafka.annotation.KafkaListener;

@Service
public class IdempotentOrderConsumer {
    
    private final ProcessedMessageRepository processedRepo;
    private final OrderRepository orderRepo;
    
    @KafkaListener(topics = "orders", groupId = "order-svc")
    @Transactional  // Database transaction ensures atomicity
    public void processOrder(String orderJson) {
        Order order = parseOrder(orderJson);
        String messageId = order.getOrderId();
        
        // IDEMPOTENCY CHECK
        if (processedRepo.existsById(messageId)) {
            log.info("Duplicate detected, skipping: {}", messageId);
            return;  // Already processed — skip!
        }
        
        // Process the order
        orderRepo.save(order);
        paymentService.charge(order);
        
        // Mark as processed (in SAME transaction)
        processedRepo.save(new ProcessedMessage(messageId, Instant.now()));
        
        // If ANY step fails → entire transaction rolls back
        // → message will be redelivered → idempotency check catches it
    }
}
```

---

## Infrastructure Examples

### Kafka Exactly-Once Configuration

```properties
# producer.properties
bootstrap.servers=localhost:9092
acks=all
enable.idempotence=true
transactional.id=my-transactional-producer
max.in.flight.requests.per.connection=5
retries=2147483647

# consumer.properties
bootstrap.servers=localhost:9092
group.id=exactly-once-consumer
isolation.level=read_committed
enable.auto.commit=false
auto.offset.reset=earliest
```

### SQS FIFO Queue (Exactly-Once Delivery)

```hcl
# Terraform — SQS FIFO Queue with deduplication
resource "aws_sqs_queue" "orders_fifo" {
  name                        = "orders.fifo"       # Must end in .fifo
  fifo_queue                  = true
  content_based_deduplication = true                # Auto-dedup by content hash
  deduplication_scope         = "messageGroup"
  fifo_throughput_limit       = "perMessageGroupId"
  
  # OR use explicit MessageDeduplicationId for custom dedup logic
}
```

```python
# Python — SQS FIFO with exactly-once delivery
import boto3, hashlib

sqs = boto3.client('sqs')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123/orders.fifo'

# Send with deduplication ID (SQS deduplicates within 5-minute window)
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='{"order_id": "ORD-123", "amount": 99.99}',
    MessageGroupId='user-456',              # Ordering within this group
    MessageDeduplicationId='ORD-123-v1'     # Same ID = dedup (within 5 min)
)

# If you resend with same MessageDeduplicationId → SQS ignores the duplicate!
```

---

## Real-World Example

### Banking — Why At-Least-Once + Idempotency is Standard

Banks CANNOT use at-most-once (might lose a $10,000 transfer). They also can't tolerate duplicates (charging someone twice). Their solution:

```
BANK TRANSFER FLOW:

1. User initiates transfer: $500 from Account A → Account B
2. Transfer Service creates event with UNIQUE ID: "TXN-789"

┌─────────────┐     ┌──────────────┐     ┌─────────────────────────┐
│  Transfer   │────▶│  Kafka Topic │────▶│  Account Service         │
│  Service    │     │  (transfers) │     │                         │
│             │     │              │     │  1. Check: "TXN-789"    │
│  TXN-789   │     │  at-least-   │     │     already processed?  │
│  $500 A→B  │     │  once        │     │                         │
└─────────────┘     └──────────────┘     │  2. NO → Execute:      │
                                         │     A: -$500            │
                                         │     B: +$500            │
                                         │     Mark TXN-789 done   │
                                         │                         │
                                         │  3. If duplicate arrives:│
                                         │     "TXN-789" exists!   │
                                         │     → SKIP (idempotent) │
                                         └─────────────────────────┘

Result: Money transferred EXACTLY ONCE, even if message delivered multiple times.
```

### Stripe — Idempotency Keys

Stripe's API uses idempotency keys to achieve exactly-once payment processing:

```
Client sends:
POST /v1/charges
Idempotency-Key: "order-12345-charge"
{amount: 5000, currency: "usd"}

First request → Stripe processes charge, stores result with key
Retry (same key) → Stripe returns CACHED result (no double-charge!)

This is at-least-once delivery + server-side idempotency = exactly-once PROCESSING
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Assuming at-most-once is "good enough" | Silent data loss in production | Use at-least-once + idempotency for critical data |
| Using auto-commit for critical consumers | Offset committed before processing → message loss | Manual commit AFTER processing |
| Not implementing idempotency | Duplicate charges, double inventory deductions | Use unique message IDs + dedup check |
| Idempotency keys without TTL | Storage grows forever | Set TTL (7 days) and handle expired keys |
| Thinking exactly-once is free | Transactions add latency + complexity | Only use where truly needed (payments, inventory) |
| Using DB auto-increment as dedup key | Different messages get different IDs even if duplicates | Use business-level IDs (order_id, txn_id) |
| Not handling the "replay" scenario | Old messages replayed after consumer reset | Idempotency must handle messages from days/weeks ago |

---

## When to Use / When NOT to Use

### Delivery Guarantee Decision Matrix:

| Scenario | Recommended | Why |
|----------|-------------|-----|
| **Real-time metrics/analytics** | At-most-once | Speed matters, losing a few data points is OK |
| **Logging** | At-most-once or at-least-once | Duplicate logs are fine; loss is usually acceptable |
| **Email notifications** | At-least-once | User getting 2 emails is better than 0 |
| **Payment processing** | At-least-once + idempotency | Must never lose, must never double-charge |
| **Inventory updates** | At-least-once + idempotency | Must never lose, duplicates handled by dedup |
| **Bank transfers** | Exactly-once (transactional) | No tolerance for loss OR duplicates |
| **Event sourcing** | Exactly-once (Kafka transactions) | Event log must be perfectly accurate |
| **Stream processing (Kafka→Kafka)** | Exactly-once (Kafka transactions) | Built-in support with read-process-write pattern |

### Cost Comparison:

```
COMPLEXITY & PERFORMANCE TRADE-OFF:

                    Complexity ──────────────────────▶
                    
     ┌──────────────┐   ┌────────────────┐   ┌──────────────────┐
     │ AT-MOST-ONCE │   │ AT-LEAST-ONCE  │   │  EXACTLY-ONCE    │
     │              │   │                │   │                  │
     │ • No retries │   │ • Retries      │   │ • Transactions   │
     │ • No ACKs    │   │ • ACKs         │   │ • Idempotency    │
     │ • No state   │   │ • DLQ          │   │ • Dedup storage  │
     │              │   │                │   │ • Coordination   │
     │ Fastest      │   │ Moderate       │   │ Slowest          │
     │ Simplest     │   │ Standard       │   │ Most Complex     │
     └──────────────┘   └────────────────┘   └──────────────────┘
     
     ◀────────────────── Performance ──────────────────────────▶
```

---

## Key Takeaways

- **At-most-once**: Fast but lossy. Use for non-critical data (metrics, logs).
- **At-least-once**: Reliable but may duplicate. The industry STANDARD for most systems. Handle duplicates with idempotency.
- **Exactly-once delivery** is theoretically impossible across network boundaries. What we achieve is **exactly-once processing** through idempotent consumers.
- **Idempotency** is the key pattern: make processing safe to repeat. Same input → same output, no matter how many times executed.
- Kafka achieves exactly-once semantics via **idempotent producers** + **transactions** (read-process-write atomically).
- SQS FIFO queues provide **exactly-once delivery** within a 5-minute deduplication window.
- In practice, most systems use **at-least-once + idempotent consumers** — it's the best balance of reliability and simplicity.

---

## What's Next?

Now that you understand delivery guarantees, you have all the building blocks for the ultimate pattern: **Event-Driven Microservices**. In the next chapter, [07-event-driven-microservices.md](./07-event-driven-microservices.md), we'll put everything together — Kafka, message queues, Pub/Sub, DLQs, and delivery guarantees — to build a complete event-driven architecture like Netflix, Uber, and Amazon use in production.
