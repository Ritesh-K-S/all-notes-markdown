# Apache Kafka — Distributed Event Streaming at Scale

> **What you'll learn**: How Apache Kafka works internally — its architecture (brokers, topics, partitions, consumer groups), why it's fundamentally different from traditional message queues, and how companies like LinkedIn, Netflix, and Uber use it to process trillions of events per day.

---

## Real-Life Analogy

Imagine a **newspaper printing press**.

Traditional message queue = a **letter**. You write it, send it to one person, they read it, it's gone.

Kafka = a **newspaper**. The newspaper is printed and stored. Anyone can read it — today, tomorrow, or next week. New subscribers can go back and read yesterday's edition. The newspaper isn't destroyed after someone reads it.

Key differences:
- **Letters (queues)**: Read once, then deleted. If you missed it, it's gone.
- **Newspaper (Kafka)**: Stored permanently (or for configured retention). Anyone can read it anytime. You can replay old editions.

Kafka is not just a message queue — it's a **distributed, durable, append-only log** that happens to support messaging.

---

## Core Concept Explained Step-by-Step

### Step 1: Kafka's Core Abstraction — The Log

Everything in Kafka is based on one simple data structure: an **append-only log**.

```
                    KAFKA TOPIC (Append-Only Log)
                    
 Offset:   0      1      2      3      4      5      6
         ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
         │ msg0 │ msg1 │ msg2 │ msg3 │ msg4 │ msg5 │ msg6 │ ← New messages
         └──────┴──────┴──────┴──────┴──────┴──────┴──────┘   appended here
                                        ▲
                                        │
                              Consumer A is HERE (offset 4)
                              Consumer B is at offset 2 (behind)
                              Consumer C is at offset 6 (caught up)
```

**Key insight**: Messages are NEVER deleted after consumption. They stay in the log for a configurable **retention period** (e.g., 7 days, 30 days, or forever). Multiple consumers can read at different speeds, and new consumers can start from the beginning.

### Step 2: Topics and Partitions

A **topic** is a category/feed name. But a topic is split into **partitions** for parallelism:

```
                        TOPIC: "orders"
                        
        Partition 0              Partition 1              Partition 2
┌───┬───┬───┬───┬───┐   ┌───┬───┬───┬───┐       ┌───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │   │ 0 │ 1 │ 2 │ 3 │       │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │
└───┴───┴───┴───┴───┘   └───┴───┴───┴───┘       └───┴───┴───┴───┴───┴───┘
  ▲                        ▲                        ▲
  │                        │                        │
Consumer A               Consumer B               Consumer C
(reads P0)               (reads P1)               (reads P2)

WHY PARTITIONS?
- Parallelism: Multiple consumers read simultaneously
- Ordering: Messages within ONE partition are strictly ordered
- Scalability: More partitions = more throughput
```

### Step 3: Brokers and Clusters

Kafka runs as a **cluster** of multiple servers called **brokers**:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        KAFKA CLUSTER                                   │
│                                                                        │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐          │
│  │  Broker 0   │      │  Broker 1   │      │  Broker 2   │          │
│  │             │      │             │      │             │          │
│  │ orders-P0   │      │ orders-P1   │      │ orders-P2   │          │
│  │ (LEADER)    │      │ (LEADER)    │      │ (LEADER)    │          │
│  │             │      │             │      │             │          │
│  │ orders-P1   │      │ orders-P2   │      │ orders-P0   │          │
│  │ (REPLICA)   │      │ (REPLICA)   │      │ (REPLICA)   │          │
│  └─────────────┘      └─────────────┘      └─────────────┘          │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │              ZooKeeper / KRaft (Coordination)                  │    │
│  │  - Leader election                                            │    │
│  │  - Cluster membership                                         │    │
│  │  - Topic configuration                                        │    │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

Each partition has:
- **One Leader**: Handles all reads and writes
- **N Replicas**: Copies on other brokers for fault tolerance
- If a leader dies, a replica is promoted to leader (automatic failover)

### Step 4: Producers — Writing to Kafka

```
┌──────────┐
│ Producer │
└────┬─────┘
     │
     │ Which partition does this message go to?
     │
     ├── Key = null         → Round-robin across partitions
     ├── Key = "user-123"   → hash(key) % num_partitions → always same partition
     └── Custom partitioner → your logic decides
     
     │
     ▼
┌──────────────────┐
│  Partition N     │
│  [append to end] │
└──────────────────┘
```

**Why keys matter**: Messages with the SAME key always go to the SAME partition. This guarantees ordering for that key. Example: All events for `user-123` land in the same partition → processed in order.

### Step 5: Consumer Groups — Scalable Consumption

```
                    TOPIC: "orders" (3 partitions)
                    
        P0                   P1                   P2
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ [m0][m1][m2]  │    │ [m0][m1][m2]  │    │ [m0][m1]      │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        │     CONSUMER GROUP: "order-processing"  │
        │                    │                    │
        ▼                    ▼                    ▼
  ┌──────────┐        ┌──────────┐        ┌──────────┐
  │Consumer 1│        │Consumer 2│        │Consumer 3│
  └──────────┘        └──────────┘        └──────────┘
  
RULES:
- Each partition is consumed by EXACTLY ONE consumer in a group
- Adding consumers = more parallelism (up to # partitions)
- If consumer dies, its partitions are reassigned (rebalance)
- Multiple consumer GROUPS can read the SAME topic independently
```

**Multiple consumer groups reading the same topic:**

```
Topic: "orders"
     │
     ├──▶ Consumer Group "payment-service"  (processes payments)
     ├──▶ Consumer Group "email-service"    (sends confirmations)
     └──▶ Consumer Group "analytics"        (tracks metrics)
     
Each group gets ALL messages (like Pub/Sub)
Within a group, messages are distributed (like a Queue)

KAFKA = Queue + Pub/Sub combined!
```

---

## How It Works Internally

### Write Path (Producer → Broker)

```
Producer                          Broker (Partition Leader)
   │                                      │
   │── 1. Connect (TCP) ────────────────▶ │
   │                                      │
   │── 2. Send ProduceRequest ──────────▶ │
   │   (topic, partition, key, value,     │
   │    timestamp, headers)               │
   │                                      │── 3. Append to segment file
   │                                      │      (sequential disk write)
   │                                      │
   │                                      │── 4. Replicate to followers
   │                                      │      (ISR = In-Sync Replicas)
   │                                      │
   │◀── 5. ACK (offset assigned) ────────│
   │                                      │
   
   acks=0  → Don't wait for ACK (fastest, data loss possible)
   acks=1  → Wait for leader ACK (balanced)
   acks=all → Wait for ALL replicas ACK (safest, slowest)
```

### Storage — Segment Files

Kafka stores data as **segment files** on disk:

```
Topic: orders, Partition: 0

Directory: /kafka-data/orders-0/
├── 00000000000000000000.log     ← Segment file (messages offset 0-999)
├── 00000000000000000000.index   ← Offset → file position mapping
├── 00000000000000000000.timeindex ← Timestamp → offset mapping
├── 00000000000000001000.log     ← Next segment (messages 1000-1999)
├── 00000000000000001000.index
└── 00000000000000001000.timeindex

Segment file format:
┌────────┬────────┬──────────┬───────────┬─────────┬─────────┐
│ Offset │  Size  │Timestamp │   Key     │  Value  │ Headers │
│ (8B)   │ (4B)  │  (8B)    │(variable) │(variable│(variable│
└────────┴────────┴──────────┴───────────┴─────────┴─────────┘
```

**Why Kafka is so fast:**
1. **Sequential I/O**: Appends only — no random seeks. Sequential disk writes can be faster than random RAM access!
2. **Zero-copy**: Uses `sendfile()` syscall — data goes from disk → network buffer without touching application memory
3. **Page cache**: OS caches recently written data in RAM automatically
4. **Batching**: Messages are batched and compressed before sending

### Consumer Offset Management

```
Consumer Group: "payment-service"

┌────────────────────────────────────────────────────┐
│        __consumer_offsets (internal topic)          │
│                                                    │
│  Group: payment-service                            │
│  ├── orders-P0: offset 1547  (last committed)     │
│  ├── orders-P1: offset 2891                        │
│  └── orders-P2: offset 988                         │
│                                                    │
│  Group: email-service                              │
│  ├── orders-P0: offset 1200  (behind!)            │
│  ├── orders-P1: offset 2891                        │
│  └── orders-P2: offset 988                         │
└────────────────────────────────────────────────────┘

Consumer tracks: "I've processed up to offset X"
If consumer restarts: picks up from last committed offset
This is how Kafka enables REPLAY — just reset offset to 0!
```

### Replication and Fault Tolerance

```
                    orders-P0 (Replication Factor = 3)
                    
Broker 0 (LEADER)        Broker 1 (FOLLOWER)      Broker 2 (FOLLOWER)
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ [0][1][2][3][4] │─────▶│ [0][1][2][3][4] │      │ [0][1][2][3]    │
│  (up to date)   │      │  (in-sync)      │      │  (slightly      │
│                 │─────▶│                 │      │   behind)       │
└─────────────────┘      └─────────────────┘      └─────────────────┘
       │                         │                         │
       │              ISR (In-Sync Replicas)               │
       │              = {Broker 0, Broker 1}               │
       │              Broker 2 is catching up              │

IF Broker 0 dies:
→ Controller detects failure
→ Broker 1 (in ISR) becomes new LEADER
→ Producers/consumers redirect to Broker 1
→ NO DATA LOSS (Broker 1 had all messages)
```

---

## Code Examples

### Python — Kafka Producer (using `confluent-kafka`)

```python
from confluent_kafka import Producer
import json

# ============ PRODUCER ============
def create_producer():
    """Create a high-throughput Kafka producer"""
    config = {
        'bootstrap.servers': 'localhost:9092',
        'acks': 'all',                    # Wait for all replicas (safest)
        'retries': 3,                     # Retry on transient errors
        'linger.ms': 5,                   # Batch messages for 5ms
        'batch.size': 16384,              # Max batch size (bytes)
        'compression.type': 'snappy',     # Compress batches
    }
    return Producer(config)


def delivery_report(err, msg):
    """Callback on message delivery (success or failure)"""
    if err:
        print(f"FAILED: {err}")
    else:
        print(f"Delivered to {msg.topic()} [{msg.partition()}] @ offset {msg.offset()}")


def publish_order_event(producer, order):
    """Publish an order event with a key (ensures ordering per user)"""
    producer.produce(
        topic='orders',
        key=order['user_id'],         # Same user → same partition → ordered!
        value=json.dumps(order),
        callback=delivery_report
    )
    producer.flush()  # Wait for delivery confirmation


# Usage
producer = create_producer()
publish_order_event(producer, {
    "user_id": "user-123",
    "order_id": "ORD-456",
    "items": ["laptop", "mouse"],
    "total": 1299.99
})
```

### Python — Kafka Consumer

```python
from confluent_kafka import Consumer, KafkaError
import json

# ============ CONSUMER ============
def create_consumer(group_id):
    """Create a consumer in a specific consumer group"""
    config = {
        'bootstrap.servers': 'localhost:9092',
        'group.id': group_id,
        'auto.offset.reset': 'earliest',  # Start from beginning if no offset
        'enable.auto.commit': False,       # Manual commit for reliability
    }
    return Consumer(config)


def process_orders():
    """Consume and process orders from Kafka"""
    consumer = create_consumer('order-processing')
    consumer.subscribe(['orders'])  # Subscribe to topic
    
    try:
        while True:
            msg = consumer.poll(timeout=1.0)  # Wait up to 1s for message
            
            if msg is None:
                continue
            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    continue  # End of partition, not an error
                print(f"Error: {msg.error()}")
                continue
            
            # Process the message
            order = json.loads(msg.value().decode('utf-8'))
            print(f"Processing order {order['order_id']} "
                  f"from partition {msg.partition()} "
                  f"at offset {msg.offset()}")
            
            # Do work here (e.g., update database)
            handle_order(order)
            
            # Commit offset AFTER successful processing
            consumer.commit(asynchronous=False)
            
    except KeyboardInterrupt:
        pass
    finally:
        consumer.close()


def handle_order(order):
    """Business logic for order processing"""
    print(f"  → Charging ${order['total']} for user {order['user_id']}")
```

### Java — Kafka Producer

```java
import org.apache.kafka.clients.producer.*;
import java.util.Properties;

public class OrderProducer {
    
    private final KafkaProducer<String, String> producer;
    
    public OrderProducer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", 
            "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", 
            "org.apache.kafka.common.serialization.StringSerializer");
        props.put("acks", "all");                // Strongest durability
        props.put("retries", 3);
        props.put("linger.ms", 5);               // Batch for 5ms
        props.put("compression.type", "snappy");
        
        this.producer = new KafkaProducer<>(props);
    }
    
    public void publishOrder(String userId, String orderJson) {
        ProducerRecord<String, String> record = new ProducerRecord<>(
            "orders",     // topic
            userId,       // key (partition by user)
            orderJson     // value
        );
        
        // Async send with callback
        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                System.err.println("Send failed: " + exception.getMessage());
            } else {
                System.out.printf("Sent to partition %d, offset %d%n",
                    metadata.partition(), metadata.offset());
            }
        });
    }
    
    public void close() { producer.close(); }
}
```

### Java — Kafka Consumer

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class OrderConsumer {
    
    public void startConsuming() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "order-processing");
        props.put("key.deserializer", 
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", 
            "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("auto.offset.reset", "earliest");
        props.put("enable.auto.commit", "false");  // Manual commit
        
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("orders"));
        
        try {
            while (true) {
                ConsumerRecords<String, String> records = 
                    consumer.poll(Duration.ofMillis(1000));
                
                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Partition=%d, Offset=%d, Key=%s%n",
                        record.partition(), record.offset(), record.key());
                    
                    // Process the order
                    processOrder(record.value());
                }
                
                // Commit after batch processing
                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }
}
```

---

## Infrastructure Examples

### Docker Compose — Full Kafka Cluster

```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka-broker-1:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_LOG_RETENTION_HOURS: 168        # 7 days
      KAFKA_LOG_SEGMENT_BYTES: 1073741824   # 1 GB segments

  kafka-broker-2:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9093

  kafka-broker-3:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9094

  # Schema Registry (for Avro/Protobuf schemas)
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on:
      - kafka-broker-1
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-broker-1:9092

  # Kafka UI for monitoring
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-broker-1:9092
```

### Kafka Topic Configuration (CLI)

```bash
# Create a topic with 6 partitions and replication factor 3
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000 \      # 7 days retention
  --config cleanup.policy=delete \        # Delete old segments
  --config min.insync.replicas=2          # At least 2 replicas must ACK

# List topics
kafka-topics.sh --list --bootstrap-server localhost:9092

# Describe topic (shows partition leaders, replicas, ISR)
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092

# Check consumer group lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-processing --describe
```

### Kubernetes — Kafka with Strimzi Operator

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-kafka
spec:
  kafka:
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
    storage:
      type: persistent-claim
      size: 500Gi
      class: gp3
    config:
      num.partitions: 6
      default.replication.factor: 3
      min.insync.replicas: 2
      log.retention.hours: 168
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 50Gi
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  labels:
    strimzi.io/cluster: production-kafka
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 604800000
    min.insync.replicas: 2
```

---

## Real-World Example

### LinkedIn — Where Kafka Was Born

Kafka was created at LinkedIn to handle their massive data pipeline:

```
LINKEDIN'S KAFKA ARCHITECTURE:

┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Profile    │  │  Messaging  │  │   Search    │  │    Ads      │
│  Views      │  │  Service    │  │  Indexing   │  │  Service    │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       ▼                ▼                ▼                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    KAFKA CLUSTER                                   │
│                                                                    │
│  7+ TRILLION messages/day                                         │
│  100+ PB of data stored                                           │
│  4000+ Kafka brokers                                              │
│                                                                    │
│  Topics: profile_views, messages, searches, ad_clicks,            │
│          page_views, connection_events, job_applications...        │
└──────────────────────────────────────────────────────────────────┘
       │                │                │                │
       ▼                ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Hadoop     │  │   Real-time │  │   People    │  │   Metrics   │
│  (batch     │  │   Analytics │  │   You May   │  │   & Alerts  │
│   analytics)│  │   (Samza)   │  │   Know      │  │             │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```

### Uber — Real-Time Trip Processing

```
UBER'S KAFKA USAGE:

Driver location update (every 4 seconds)
        │
        ▼
┌───────────────────────────────────────────────┐
│   KAFKA: "driver_locations" topic              │
│   Partitioned by city/region                  │
│   ~1 million events/second                    │
└───────────────┬───────────────────────────────┘
                │
    ┌───────────┼───────────────────┐
    ▼           ▼                   ▼
┌────────┐  ┌──────────┐     ┌──────────────┐
│ Rider  │  │   ETA    │     │  Surge       │
│Matching│  │Prediction│     │  Pricing     │
│Service │  │ Service  │     │  Calculator  │
└────────┘  └──────────┘     └──────────────┘

Uber processes 1+ trillion Kafka messages/day!
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Too few partitions | Can't parallelize beyond partition count | Start with 6-12 partitions; increase based on throughput needs |
| Too many partitions | More memory, longer leader elections, more open files | Rule of thumb: #partitions × #topics < 10,000 per broker |
| Not using message keys | No ordering guarantee for related events | Use entity ID (user_id, order_id) as key |
| `acks=0` in production | Data loss when broker crashes | Use `acks=all` + `min.insync.replicas=2` for critical data |
| Auto-commit offsets | Processing fails after commit → lost message | Manual commit AFTER successful processing |
| Consumer group with more consumers than partitions | Extra consumers sit idle (wasted resources) | #consumers ≤ #partitions |
| No monitoring on consumer lag | Consumers fall behind, data becomes stale | Alert when lag > threshold (e.g., 10,000 messages) |
| Unbounded message size | Broker OOM, slow replication | Cap at 1MB; store large payloads externally (S3) |
| Single partition for ordered events | Throughput limited to one consumer | Use finer-grained keys + enough partitions |

---

## When to Use / When NOT to Use

### ✅ Use Kafka When:

- **High throughput** needed (>100K messages/second)
- **Event replay** required (consumers need to reprocess old data)
- **Multiple consumer groups** reading the same stream independently
- **Event sourcing** architecture (Kafka IS the source of truth)
- **Stream processing** (Kafka Streams, Flink, Spark Streaming)
- **Long-term event storage** (retention days/weeks/forever)
- **Strict ordering** within a partition is needed
- **Decoupling** many producers from many consumers

### ❌ Don't Use Kafka When:

- **Low throughput, simple queuing** (use RabbitMQ or SQS — less operational overhead)
- **Request-response pattern** (Kafka is not designed for RPC)
- **Complex routing** (RabbitMQ's exchanges are more flexible for routing)
- **Small team, simple system** (Kafka has significant operational complexity)
- **Sub-millisecond latency** required (Kafka adds 2-10ms latency)
- **Exactly-once is critical with simple setup** (possible but complex)

### Kafka vs RabbitMQ vs SQS:

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| **Throughput** | Millions/sec | Tens of thousands/sec | Unlimited (managed) |
| **Message retention** | Days/weeks/forever | Until consumed | Up to 14 days |
| **Replay** | ✅ Yes (offset reset) | ❌ No | ❌ No |
| **Ordering** | Per partition | Per queue | FIFO queues only |
| **Consumer groups** | ✅ Native | Manual setup | ❌ No |
| **Routing** | Topic/partition only | Exchanges, bindings, patterns | Queue-based |
| **Operational cost** | High (run cluster) | Medium | None (managed) |
| **Best for** | Event streaming, big data | Task queues, routing | Simple cloud queuing |

---

## Key Takeaways

- Kafka is a **distributed commit log**, not just a message queue. Messages persist for days/weeks, enabling replay.
- **Topics** are split into **partitions** for parallelism. Order is guaranteed ONLY within a partition.
- **Consumer groups** provide both queue semantics (within group) and pub/sub (across groups).
- **Keys** determine partition assignment — same key = same partition = ordered processing.
- Kafka achieves incredible throughput via **sequential I/O**, **zero-copy**, **batching**, and **compression**.
- **Replication** (ISR) provides fault tolerance — if a broker dies, data is safe on replicas.
- Use `acks=all` + `min.insync.replicas=2` for production workloads where data loss is unacceptable.

---

## What's Next?

What happens when a message can't be processed — the consumer keeps crashing, the data is malformed, or a downstream service is permanently down? You can't just lose that message. In the next chapter, [05-dead-letter-queues.md](./05-dead-letter-queues.md), we'll explore **Dead Letter Queues (DLQs)** — the safety net that catches failed messages and gives you a second chance.
