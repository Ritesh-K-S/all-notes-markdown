# Event-Driven Architecture (EDA)

> **What you'll learn**: How systems communicate by producing and consuming events instead of direct calls, enabling loose coupling, real-time processing, and extreme scalability.

---

## Real-Life Analogy

Imagine a **newspaper publishing system**:

The newspaper **doesn't know who reads it**. It simply publishes the morning edition and anyone who has a subscription gets a copy. The newspaper doesn't call each reader individually — it publishes once, and interested readers consume it.

Now compare this with a **phone call**:
- **Phone call** (synchronous) = "Hey John, here's the news" → waits for John to respond → calls Mary → waits again
- **Newspaper** (event-driven) = "Here's what happened" → published once → everyone who cares gets it simultaneously

That's the difference between **request-response** and **event-driven** architecture. Instead of Service A calling Service B directly, Service A says "something happened" (publishes an event), and any interested service reacts to it.

---

## Core Concept Explained Step-by-Step

### What is Event-Driven Architecture?

**EDA** is a design pattern where the flow of the system is determined by **events** — significant changes in state that are produced, detected, and consumed by different parts of the system.

```
TRADITIONAL (Request-Response):          EVENT-DRIVEN:

┌──────────┐  "Create order"  ┌────────┐       ┌──────────┐  "OrderCreated"  ┌────────────┐
│  Order   │────────────────▶│Inventory│       │  Order   │────────────────▶│   Broker   │
│ Service  │◀────────────────│ Service │       │ Service  │                 │(Kafka/SQS) │
└──────────┘  "Stock updated" └────────┘       └──────────┘                 └─────┬──────┘
                                                                                   │
      Order Service KNOWS about                                    ┌───────────────┼──────────────┐
      Inventory Service                                            ▼               ▼              ▼
      (tight coupling)                                      ┌──────────┐    ┌────────────┐  ┌─────────┐
                                                            │Inventory │    │Notification│  │Analytics│
                                                            │ Service  │    │  Service   │  │ Service │
                                                            └──────────┘    └────────────┘  └─────────┘

                                                            Order Service has NO IDEA who
                                                            listens to its events (loose coupling)
```

### Core Components

| Component | Role | Example |
|---|---|---|
| **Event** | A record that something happened | `OrderCreated`, `PaymentReceived`, `UserSignedUp` |
| **Producer** | Creates and publishes events | Order Service publishes `OrderCreated` |
| **Consumer** | Subscribes to and processes events | Email Service reacts to `OrderCreated` |
| **Broker** | Middleware that routes events | Kafka, RabbitMQ, AWS SNS/SQS |
| **Channel/Topic** | Named stream where events flow | `orders` topic, `payments` topic |

### What is an Event?

An event is an **immutable fact** — something that happened in the past. It's NOT a command (do something) — it's a notification (something happened).

```json
{
  "event_type": "OrderCreated",
  "event_id": "evt-123-abc",
  "timestamp": "2024-01-15T10:30:00Z",
  "source": "order-service",
  "data": {
    "order_id": "ORD-001",
    "customer_id": "CUST-42",
    "items": [
      {"product_id": "PROD-99", "quantity": 2, "price": 29.99}
    ],
    "total": 59.98
  }
}
```

---

## How It Works Internally

### Event Flow Patterns

**Pattern 1: Simple Event Notification**
```
Producer publishes event with minimal data.
Consumer calls back to get full details if needed.

Order Service ──publish──▶ { "event": "OrderCreated", "order_id": "ORD-001" }
                                          │
                                          ▼
Notification Service ──────────────▶ GET /orders/ORD-001 (fetches details)
                                          │
                                          ▼
                                    Send email with order details
```

**Pattern 2: Event-Carried State Transfer**
```
Producer publishes event WITH all necessary data.
Consumer doesn't need to call back.

Order Service ──publish──▶ { "event": "OrderCreated",
                             "order_id": "ORD-001",
                             "customer_email": "alice@mail.com",
                             "items": [...],
                             "total": 59.98 }
                                          │
                                          ▼
Notification Service ──────▶ Send email directly (no callback needed)
```

**Pattern 3: Event Sourcing (covered in Chapter 4.8)**
```
Store ALL state changes as a sequence of events.
Current state = replay all events from the beginning.
```

### Pub/Sub vs Event Streaming

```
PUB/SUB (RabbitMQ, SNS):                EVENT STREAMING (Kafka):

Messages are consumed and              Events are STORED in a log.
DELETED after processing.              Consumers read at their own pace.
                                        Can replay from any point.

┌─────────┐     ┌───────┐              ┌─────────┐     ┌───────────────┐
│Producer │────▶│ Queue │              │Producer │────▶│  Topic Log    │
└─────────┘     └───┬───┘              └─────────┘     │ [e1][e2][e3]  │
                    │                                   │ [e4][e5][e6]  │
                    ▼ (message deleted                   └───┬───────┬──┘
              ┌──────────┐  after ack)                      │       │
              │ Consumer │                                  ▼       ▼
              └──────────┘                           Consumer A  Consumer B
                                                    (offset: 3) (offset: 5)
                                                     Can go back!
```

### The Event Broker Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    EVENT BROKER (Kafka)                         │
│                                                                │
│  Topic: "orders"              Topic: "payments"                │
│  ┌─────────────────────┐     ┌─────────────────────┐          │
│  │ Partition 0:         │     │ Partition 0:         │          │
│  │ [evt1][evt2][evt3]   │     │ [evt1][evt2]         │          │
│  │ Partition 1:         │     │ Partition 1:         │          │
│  │ [evt4][evt5][evt6]   │     │ [evt3][evt4][evt5]   │          │
│  │ Partition 2:         │     └─────────────────────┘          │
│  │ [evt7][evt8]         │                                      │
│  └─────────────────────┘     Topic: "notifications"            │
│                               ┌─────────────────────┐          │
│                               │ [evt1][evt2][evt3]   │          │
│                               └─────────────────────┘          │
└────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python (Kafka Producer & Consumer)

```python
# order_service/events.py — Publishing events when orders are created
from confluent_kafka import Producer
import json, uuid
from datetime import datetime

producer = Producer({'bootstrap.servers': 'kafka:9092'})

def publish_order_created(order):
    """Publish an event when a new order is created."""
    event = {
        "event_id": str(uuid.uuid4()),
        "event_type": "OrderCreated",
        "timestamp": datetime.utcnow().isoformat(),
        "source": "order-service",
        "data": {
            "order_id": order.id,
            "customer_id": order.customer_id,
            "customer_email": order.customer_email,
            "items": order.items,
            "total": order.total
        }
    }
    # Publish to "orders" topic, partitioned by customer_id
    producer.produce(
        topic="orders",
        key=order.customer_id.encode(),
        value=json.dumps(event).encode(),
        callback=delivery_report
    )
    producer.flush()

# notification_service/consumer.py — Reacting to order events
from confluent_kafka import Consumer

consumer = Consumer({
    'bootstrap.servers': 'kafka:9092',
    'group.id': 'notification-service',
    'auto.offset.reset': 'earliest'
})
consumer.subscribe(['orders'])

def process_events():
    """Listen for order events and send notifications."""
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        
        event = json.loads(msg.value().decode())
        
        if event["event_type"] == "OrderCreated":
            send_order_confirmation_email(
                to=event["data"]["customer_email"],
                order_id=event["data"]["order_id"],
                total=event["data"]["total"]
            )
        elif event["event_type"] == "OrderShipped":
            send_shipping_notification(event["data"])
        
        consumer.commit()  # Mark as processed
```

### Java (Spring Boot with Kafka)

```java
// OrderService.java — Produces events
@Service
public class OrderService {
    
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    @Autowired private OrderRepository orderRepo;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = orderRepo.save(new Order(request));
        
        // Publish event — we don't know or care who's listening
        OrderEvent event = new OrderEvent(
            UUID.randomUUID().toString(),
            "OrderCreated",
            Instant.now(),
            new OrderData(order.getId(), order.getCustomerId(), order.getTotal())
        );
        kafkaTemplate.send("orders", order.getCustomerId(), event);
        
        return order;
    }
}

// NotificationConsumer.java — Consumes events
@Component
public class NotificationConsumer {
    
    @Autowired private EmailService emailService;
    
    @KafkaListener(topics = "orders", groupId = "notification-service")
    public void handleOrderEvent(OrderEvent event) {
        switch (event.getType()) {
            case "OrderCreated":
                emailService.sendOrderConfirmation(
                    event.getData().getCustomerEmail(),
                    event.getData().getOrderId()
                );
                break;
            case "OrderShipped":
                emailService.sendShippingNotification(event.getData());
                break;
        }
    }
}

// InventoryConsumer.java — Same event, different reaction
@Component
public class InventoryConsumer {
    
    @Autowired private InventoryRepository inventoryRepo;
    
    @KafkaListener(topics = "orders", groupId = "inventory-service")
    public void handleOrderEvent(OrderEvent event) {
        if ("OrderCreated".equals(event.getType())) {
            // Reduce stock for each ordered item
            for (OrderItem item : event.getData().getItems()) {
                inventoryRepo.decrementStock(item.getProductId(), item.getQuantity());
            }
        }
    }
}
```

---

## Infrastructure Example

### Kafka Cluster Setup

```yaml
# docker-compose.yml — Kafka infrastructure
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka-1:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3

  kafka-2:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092

  kafka-3:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092

  # Schema Registry for event contract validation
  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-1:9092
```

### Event-Driven System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION EDA SYSTEM                             │
│                                                                      │
│   PRODUCERS:                    KAFKA CLUSTER:        CONSUMERS:     │
│                                                                      │
│ ┌───────────┐                 ┌─────────────┐    ┌──────────────┐   │
│ │   Order   │──publish───────▶│   orders    │───▶│  Inventory   │   │
│ │  Service  │                 │   topic     │───▶│  Notification│   │
│ └───────────┘                 └─────────────┘───▶│  Analytics   │   │
│                                                  └──────────────┘   │
│ ┌───────────┐                 ┌─────────────┐    ┌──────────────┐   │
│ │  Payment  │──publish───────▶│  payments   │───▶│  Order Svc   │   │
│ │  Service  │                 │   topic     │───▶│  Accounting  │   │
│ └───────────┘                 └─────────────┘    └──────────────┘   │
│                                                                      │
│ ┌───────────┐                 ┌─────────────┐    ┌──────────────┐   │
│ │   User    │──publish───────▶│   users     │───▶│  Marketing   │   │
│ │  Service  │                 │   topic     │───▶│  Analytics   │   │
│ └───────────┘                 └─────────────┘    └──────────────┘   │
│                                                                      │
│                         ┌──────────────────────┐                    │
│                         │  Schema Registry     │                    │
│                         │  (event contracts)   │                    │
│                         └──────────────────────┘                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Uber — Rides are Event-Driven

```
RIDER REQUESTS RIDE:
┌──────────┐  "RideRequested"   ┌─────────────┐
│  Rider   │───────────────────▶│    Kafka    │
│   App    │                    └──────┬──────┘
└──────────┘                           │
                          ┌────────────┼────────────────┐
                          ▼            ▼                ▼
                   ┌──────────┐ ┌──────────┐    ┌──────────┐
                   │ Matching │ │  Pricing │    │ Dispatch │
                   │ Service  │ │  Service │    │  Service │
                   │(find     │ │(calc fare│    │(find     │
                   │ driver)  │ │ + surge) │    │ nearest) │
                   └────┬─────┘ └──────────┘    └──────────┘
                        │
                        ▼ "DriverAssigned"
                   ┌──────────┐
                   │  Kafka   │
                   └────┬─────┘
                        │
              ┌─────────┼─────────┐
              ▼                   ▼
       ┌──────────┐        ┌──────────┐
       │  Rider   │        │  Driver  │
       │   App    │        │   App    │
       │ (notify) │        │ (navigate│
       └──────────┘        └──────────┘
```

### LinkedIn — 7 Trillion Events/Day

LinkedIn processes **7+ trillion events per day** through Kafka:
- Profile updates → Search indexing
- Connection requests → Notification system
- Content posts → Feed generation
- Job applications → Recruiter notifications

### Amazon — EventBridge

Amazon uses its own service (EventBridge) to connect **hundreds of microservices** through events:
- `ItemAddedToCart` → Recommendation engine, inventory check, analytics
- `OrderPlaced` → Payment processing, warehouse, shipping, email
- `PaymentFailed` → Retry logic, customer notification, fraud detection

---

## Common Mistakes / Pitfalls

### 1. Event Order Dependency
❌ **Mistake**: Assuming events arrive in the same order they were published.
✅ **Fix**: Design consumers to be order-independent, OR use Kafka partitions (same partition = ordered).

### 2. No Event Schema Management
❌ **Mistake**: Producer changes event format, breaking all consumers.
✅ **Fix**: Use Schema Registry (Avro/Protobuf) with backward-compatible evolution rules.

### 3. Distributed Monolith via Events
❌ **Mistake**: Creating chains where Service A's event triggers B, which triggers C, which triggers D — tightly coupled sequence.
✅ **Fix**: Events should represent facts, not commands. If you need orchestration, use a Saga (Chapter 13.6).

```
BAD (Event Chain = Distributed Monolith):
A ──▶ B ──▶ C ──▶ D ──▶ E    (if C fails, everything stalls)

GOOD (Event Fan-Out):
A ──▶ B
  ──▶ C     (each consumer is independent)
  ──▶ D
  ──▶ E
```

### 4. Not Handling Duplicate Events
❌ **Mistake**: Processing the same event twice leads to double charges or double emails.
✅ **Fix**: Make consumers **idempotent** — processing the same event twice produces the same result (see Chapter 12.7).

### 5. Losing Events
❌ **Mistake**: Events disappear when a consumer crashes mid-processing.
✅ **Fix**: Use at-least-once delivery + idempotent consumers. Commit offset only AFTER successful processing.

---

## When to Use / When NOT to Use

### ✅ Use EDA When:

| Criteria | Why |
|---|---|
| **Loose coupling between services** | Producer doesn't know/care about consumers |
| **Multiple consumers for same event** | Publish once, many react |
| **Real-time processing needed** | React to events as they happen |
| **High throughput** | Kafka handles millions of events/second |
| **Audit trail required** | Event log = complete history of what happened |
| **Extensibility** | Add new consumers without modifying producers |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Need immediate response** | Events are async — no instant reply |
| **Simple CRUD operations** | Over-engineering for basic read/write |
| **Strong consistency required** | Eventual consistency is inherent in EDA |
| **Small system (< 3 services)** | Direct calls are simpler |
| **Team inexperienced with async** | Debugging async flows is much harder |

---

## Key Takeaways

- 📢 **Events are facts** — "OrderCreated" is something that happened. It's not a command to do something.
- 🔌 **Producers and consumers are decoupled** — the producer doesn't know who consumes its events.
- 📬 **Event brokers** (Kafka, RabbitMQ, SNS/SQS) route events from producers to consumers.
- ⚡ **EDA enables real-time reactions** — as soon as something happens, interested services react.
- 🔄 **Eventual consistency is the trade-off** — consumers process events asynchronously, so data isn't immediately consistent everywhere.
- 🛡️ **Idempotent consumers are critical** — events can be delivered more than once. Processing them twice must be safe.
- 📈 **EDA scales massively** — LinkedIn processes 7T events/day. Kafka can handle millions of events per second per partition.

---

## What's Next?

Event-driven architecture opens the door to powerful patterns for managing complex data. In **Chapter 4.7: CQRS (Command Query Responsibility Segregation)**, we'll learn how to separate read operations from write operations — optimizing each independently for performance and scalability.
