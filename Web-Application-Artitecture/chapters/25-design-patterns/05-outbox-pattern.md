# Outbox Pattern — Reliable Event Publishing

> **What you'll learn**: How to solve the "dual write" problem — ensuring that when your service updates its database AND publishes an event, both always happen together (or neither does), even in the face of crashes and network failures.

---

## Real-Life Analogy

Imagine you're a cashier at a store. When a customer pays:
1. You update the cash register (record the sale)
2. You give the customer a receipt

What if you update the register but the receipt printer jams? The customer leaves without proof of payment. What if you print the receipt but forget to record it? The accounting is wrong.

The **Outbox Pattern** is like writing the receipt INTO the register's transaction log. The register and receipt are recorded in ONE action. A separate "receipt delivery person" periodically checks the register and ensures all receipts are delivered. Even if the delivery person crashes, they pick up where they left off because the receipts are safely in the register.

---

## The Problem: Dual Writes

In microservices, a common pattern is:

```
Service receives request
    │
    ├── 1. Update Database     (e.g., save order)
    │
    └── 2. Publish Event       (e.g., "OrderCreated" to Kafka)
```

This looks simple, but it has a **fatal flaw**:

```
SCENARIO A: DB succeeds, Kafka fails
─────────────────────────────────────
1. Save order to DB          ✅ Success
2. Publish to Kafka          ❌ Kafka is down!
   Result: Order exists but no event published.
   Other services never know about it!

SCENARIO B: Kafka succeeds, DB fails
─────────────────────────────────────
1. Publish to Kafka          ✅ Success
2. Save order to DB          ❌ DB transaction rolls back!
   Result: Event published but order doesn't exist.
   Other services process a phantom order!

SCENARIO C: Service crashes between steps
─────────────────────────────────────
1. Save order to DB          ✅ Success
2. --- SERVICE CRASHES ---
3. Publish to Kafka          ❌ Never happens!
```

**You cannot atomically update two different systems** (database + message broker). This is the "dual write" problem.

---

## The Solution: Transactional Outbox

Instead of publishing directly to the message broker, write the event to an **outbox table** in the SAME database transaction as your business data:

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE OUTBOX PATTERN                             │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              SINGLE DATABASE TRANSACTION                  │    │
│  │                                                           │    │
│  │  1. INSERT INTO orders (...)                              │    │
│  │  2. INSERT INTO outbox (event_type, payload, ...)         │    │
│  │                                                           │    │
│  │  COMMIT;  ← Both happen atomically or neither does!       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│                          │                                        │
│                          ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │          OUTBOX RELAY / POLLER                            │    │
│  │  (Separate process that reads outbox and publishes)       │    │
│  │                                                           │    │
│  │  1. SELECT * FROM outbox WHERE published = false          │    │
│  │  2. Publish each event to Kafka/RabbitMQ                  │    │
│  │  3. Mark as published (or delete from outbox)             │    │
│  │                                                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                        │
│                          ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              MESSAGE BROKER (Kafka, RabbitMQ)             │    │
│  │                                                           │    │
│  │  Events reliably delivered to consumers                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Step by Step

```
Step 1: Business logic + outbox write (ATOMIC)
══════════════════════════════════════════════

BEGIN TRANSACTION;
  INSERT INTO orders (id, customer_id, total, status)
  VALUES ('ord_123', 'cust_456', 99.99, 'CREATED');
  
  INSERT INTO outbox (id, event_type, aggregate_id, payload, created_at)
  VALUES ('evt_789', 'OrderCreated', 'ord_123', 
          '{"order_id":"ord_123","total":99.99}', NOW());
COMMIT;

✅ If COMMIT succeeds: order AND event are both saved.
✅ If COMMIT fails: neither is saved. Consistent!


Step 2: Relay publishes event (separate process)
═════════════════════════════════════════════════

Outbox Relay (running every 100ms or via CDC):
  1. Read: SELECT * FROM outbox WHERE published = false 
           ORDER BY created_at LIMIT 100;
  2. Publish each to Kafka
  3. Update: UPDATE outbox SET published = true WHERE id IN (...)
  
  (Or delete: DELETE FROM outbox WHERE id IN (...))


Step 3: Consumers process event
═══════════════════════════════

Inventory Service reads "OrderCreated" from Kafka → reserves stock
Payment Service reads "OrderCreated" from Kafka → charges customer
```

---

## The Outbox Table Schema

```sql
CREATE TABLE outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(255) NOT NULL,    -- e.g., 'Order'
    aggregate_id    VARCHAR(255) NOT NULL,    -- e.g., 'ord_123'
    event_type      VARCHAR(255) NOT NULL,    -- e.g., 'OrderCreated'
    payload         JSONB NOT NULL,           -- Event data
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMP NULL,
    retry_count     INT NOT NULL DEFAULT 0
);

-- Index for the relay to efficiently find unpublished events
CREATE INDEX idx_outbox_unpublished ON outbox (published, created_at)
    WHERE published = FALSE;
```

---

## Two Approaches to Reading the Outbox

### Approach 1: Polling (Simple)

```
┌──────────────┐                    ┌───────────────┐
│   Outbox     │  poll every 100ms  │  Outbox       │
│   Table      │◀───────────────────│  Relay/Poller │
│              │                    │               │
│  event 1 ✓  │                    │  Read → Pub   │
│  event 2 ✗  │───publish──────────▶  to Kafka    │
│  event 3 ✗  │                    │               │
└──────────────┘                    └───────────────┘

Pros: Simple to implement
Cons: Slight delay (polling interval), DB load from frequent queries
```

### Approach 2: Change Data Capture / CDC (Advanced)

```
┌──────────────┐                    ┌───────────────┐
│   Outbox     │  DB transaction    │   Debezium    │
│   Table      │  log streaming     │   (CDC)       │
│              │──────────────────▶ │               │
│  PostgreSQL  │  WAL (Write-Ahead  │  Reads DB log │
│  WAL log     │  Log) captured     │  Publishes to │
│              │                    │  Kafka        │
└──────────────┘                    └───────────────┘

Pros: Near real-time, no polling overhead, no extra DB queries
Cons: More complex infrastructure (Debezium, Kafka Connect)
```

---

## Code Examples

### Python: Outbox Pattern with SQLAlchemy

```python
import uuid
import json
from datetime import datetime
from sqlalchemy import create_engine, Column, String, Boolean, DateTime, Text
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy.dialects.postgresql import UUID, JSONB

Base = declarative_base()

# ─── Outbox Table Model ──────────────────────────────────
class OutboxEvent(Base):
    __tablename__ = "outbox"
    
    id = Column(UUID, primary_key=True, default=uuid.uuid4)
    aggregate_type = Column(String(255), nullable=False)
    aggregate_id = Column(String(255), nullable=False)
    event_type = Column(String(255), nullable=False)
    payload = Column(JSONB, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    published = Column(Boolean, default=False)

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(String(50), primary_key=True)
    customer_id = Column(String(50), nullable=False)
    total = Column(String(20), nullable=False)
    status = Column(String(20), default="CREATED")


# ─── Service that writes to both tables atomically ────────
class OrderService:
    def __init__(self, session_factory):
        self.session_factory = session_factory
    
    def create_order(self, customer_id: str, items: list, total: float):
        session = self.session_factory()
        
        try:
            order_id = f"ord_{uuid.uuid4().hex[:12]}"
            
            # Business data
            order = Order(
                id=order_id,
                customer_id=customer_id,
                total=str(total),
                status="CREATED"
            )
            session.add(order)
            
            # Outbox event (SAME transaction!)
            event = OutboxEvent(
                aggregate_type="Order",
                aggregate_id=order_id,
                event_type="OrderCreated",
                payload={
                    "order_id": order_id,
                    "customer_id": customer_id,
                    "items": items,
                    "total": total,
                    "status": "CREATED"
                }
            )
            session.add(event)
            
            # ATOMIC COMMIT — both or neither
            session.commit()
            return order_id
            
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()


# ─── Outbox Relay (separate process/thread) ───────────────
from kafka import KafkaProducer
import time

class OutboxRelay:
    def __init__(self, session_factory, kafka_producer):
        self.session_factory = session_factory
        self.producer = kafka_producer
    
    def poll_and_publish(self):
        """Called periodically (every 100ms-1s)"""
        session = self.session_factory()
        
        try:
            # Fetch unpublished events
            events = session.query(OutboxEvent)\
                .filter(OutboxEvent.published == False)\
                .order_by(OutboxEvent.created_at)\
                .limit(100)\
                .all()
            
            for event in events:
                # Publish to Kafka
                topic = f"{event.aggregate_type.lower()}-events"
                self.producer.send(
                    topic,
                    key=event.aggregate_id.encode(),
                    value=json.dumps(event.payload).encode()
                )
                
                # Mark as published
                event.published = True
            
            session.commit()
            
        except Exception as e:
            session.rollback()
            print(f"Outbox relay error: {e}")
        finally:
            session.close()
    
    def run(self, interval_ms=500):
        """Run the relay loop"""
        while True:
            self.poll_and_publish()
            time.sleep(interval_ms / 1000)
```

### Java: Outbox Pattern with Spring Boot + JPA

```java
// ─── Outbox Entity ───────────────────────────────────────
@Entity
@Table(name = "outbox")
public class OutboxEvent {
    @Id
    @GeneratedValue
    private UUID id;
    
    @Column(nullable = false)
    private String aggregateType;
    
    @Column(nullable = false)
    private String aggregateId;
    
    @Column(nullable = false)
    private String eventType;
    
    @Column(columnDefinition = "jsonb", nullable = false)
    private String payload;
    
    @Column(nullable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
    
    @Column(nullable = false)
    private boolean published = false;
    
    // constructors, getters, setters...
}

// ─── Order Service (atomic write) ────────────────────────
@Service
public class OrderService {
    
    private final OrderRepository orderRepo;
    private final OutboxRepository outboxRepo;
    
    @Transactional  // ← SAME transaction for both writes!
    public String createOrder(CreateOrderRequest request) {
        String orderId = "ord_" + UUID.randomUUID().toString().substring(0, 12);
        
        // Save business entity
        Order order = new Order(orderId, request.customerId(), 
            request.total(), OrderStatus.CREATED);
        orderRepo.save(order);
        
        // Save outbox event (SAME transaction!)
        OutboxEvent event = new OutboxEvent();
        event.setAggregateType("Order");
        event.setAggregateId(orderId);
        event.setEventType("OrderCreated");
        event.setPayload(toJson(Map.of(
            "order_id", orderId,
            "customer_id", request.customerId(),
            "total", request.total(),
            "items", request.items()
        )));
        outboxRepo.save(event);
        
        return orderId;
    }
}

// ─── Outbox Relay (scheduled task) ───────────────────────
@Component
@Slf4j
public class OutboxRelay {
    
    private final OutboxRepository outboxRepo;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedDelay = 500)  // Run every 500ms
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxRepo
            .findByPublishedFalseOrderByCreatedAt(PageRequest.of(0, 100));
        
        for (OutboxEvent event : events) {
            try {
                String topic = event.getAggregateType().toLowerCase() + "-events";
                
                kafkaTemplate.send(topic, event.getAggregateId(), event.getPayload())
                    .get(5, TimeUnit.SECONDS);  // Wait for ack
                
                event.setPublished(true);
                log.debug("Published event: {} for {}", 
                    event.getEventType(), event.getAggregateId());
                    
            } catch (Exception e) {
                log.error("Failed to publish event {}: {}", 
                    event.getId(), e.getMessage());
                // Will retry on next poll
                break;  // Stop to maintain ordering
            }
        }
    }
}
```

---

## CDC-Based Outbox with Debezium

Instead of polling, use **Debezium** to capture database changes in real-time:

```
┌───────────────────────────────────────────────────────────────┐
│                   CDC-BASED OUTBOX FLOW                        │
│                                                                │
│  Application                                                   │
│    │                                                           │
│    │ INSERT INTO outbox (...)                                  │
│    ▼                                                           │
│  ┌────────────────────┐                                        │
│  │    PostgreSQL       │                                        │
│  │                     │                                        │
│  │  orders table       │                                        │
│  │  outbox table  ─────┼──── WAL (Write-Ahead Log)            │
│  └────────────────────┘         │                              │
│                                  │ streams changes              │
│                                  ▼                              │
│                        ┌────────────────────┐                  │
│                        │    Debezium         │                  │
│                        │  (Kafka Connect)    │                  │
│                        └─────────┬──────────┘                  │
│                                  │ publishes                    │
│                                  ▼                              │
│                        ┌────────────────────┐                  │
│                        │   Apache Kafka      │                  │
│                        │                     │                  │
│                        │  outbox.events topic│                  │
│                        └─────────┬──────────┘                  │
│                                  │                              │
│                     ┌────────────┼────────────┐                │
│                     ▼            ▼            ▼                │
│               ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│               │Inventory │ │ Payment  │ │Notification│         │
│               │ Service  │ │ Service  │ │  Service  │          │
│               └──────────┘ └──────────┘ └──────────┘          │
└───────────────────────────────────────────────────────────────┘
```

### Debezium Connector Configuration:

```json
{
    "name": "outbox-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres",
        "database.port": "5432",
        "database.user": "dbuser",
        "database.password": "dbpass",
        "database.dbname": "orderservice",
        "table.include.list": "public.outbox",
        "transforms": "outbox",
        "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
        "transforms.outbox.table.field.event.key": "aggregate_id",
        "transforms.outbox.table.field.event.type": "event_type",
        "transforms.outbox.table.field.event.payload": "payload",
        "transforms.outbox.route.topic.replacement": "${routedByValue}-events"
    }
}
```

---

## Ordering Guarantees

```
┌─────────────────────────────────────────────────────────────┐
│              EVENT ORDERING CONSIDERATIONS                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Events for the SAME aggregate must be ordered:              │
│                                                               │
│  Order ord_123:                                               │
│    1. OrderCreated   ← Must come first                       │
│    2. OrderConfirmed ← Must come second                      │
│    3. OrderShipped   ← Must come third                       │
│                                                               │
│  Solution: Use aggregate_id as Kafka partition key            │
│  → All events for ord_123 go to same partition                │
│  → Same partition = guaranteed ordering                       │
│                                                               │
│  Events for DIFFERENT aggregates don't need ordering:        │
│    ord_123: OrderCreated  ]                                  │
│    ord_456: OrderCreated  ] ← Can arrive in any order        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Debezium at Major Companies

- **Airbnb**: Uses CDC-based outbox for their payment event system. When a booking payment is recorded, the event reliably flows to notification, analytics, and fraud detection services.

- **Wepay (Chase)**: Built their entire event-driven architecture on the outbox pattern with Debezium. Every financial transaction generates reliable events.

- **Zalando**: Their event-driven microservices use the outbox pattern extensively. They open-sourced "Nakadi" — their event broker that builds on these patterns.

### The Pattern at Scale

```
At scale (millions of events/day):

┌─────────────────────────────────────────────────────────────┐
│  Service A: 1000 writes/sec → 1000 outbox events/sec        │
│  Service B: 500 writes/sec  → 500 outbox events/sec         │
│  Service C: 2000 writes/sec → 2000 outbox events/sec        │
│                                                               │
│  Total: 3500 events/sec flowing through Kafka                │
│  With CDC: <100ms latency from DB write to event delivery    │
│  With Polling (500ms interval): ~500ms avg latency           │
└─────────────────────────────────────────────────────────────┘
```

---

## Outbox Cleanup

The outbox table grows continuously. You need a cleanup strategy:

```sql
-- Option 1: Delete published events older than 7 days
DELETE FROM outbox 
WHERE published = true 
AND created_at < NOW() - INTERVAL '7 days';

-- Option 2: Partition by date and drop old partitions
CREATE TABLE outbox (
    ...
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE outbox_2024_01 PARTITION OF outbox 
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Drop old partitions (instant, no row-by-row delete)
DROP TABLE outbox_2023_11;
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Publishing directly to Kafka without outbox | Dual write problem — data inconsistency | Always use outbox for reliability |
| Not maintaining event order | Consumers see events out of sequence | Use aggregate_id as partition key |
| Outbox table grows forever | DB storage exhaustion, slow queries | Schedule cleanup of published events |
| Relay processes same event twice | Duplicate processing downstream | Make consumers idempotent (check event ID) |
| Polling too frequently | DB load from constant queries | Use CDC, or poll every 500ms-1s |
| Polling too infrequently | High latency for event delivery | Balance: 500ms is usually good |
| Forgetting to index outbox table | Slow queries when outbox is large | Index on (published, created_at) |
| Large payloads in outbox | DB bloat, slow writes | Store minimal data; consumers can fetch details |

---

## When to Use / When NOT to Use

### Use the Outbox Pattern When:
- Service needs to update DB AND publish events reliably
- You're implementing sagas across microservices
- Event ordering matters (use aggregate_id as key)
- You cannot tolerate lost events (financial, order processing)
- You're using a relational database

### May Not Need It When:
- Using an event store as primary storage (event sourcing — events ARE the data)
- Simple fire-and-forget notifications (losing one email notification is acceptable)
- Synchronous request-response patterns (no events needed)
- Using a database that natively supports change streams (MongoDB Change Streams can work directly)

---

## Key Takeaways

1. **The dual write problem is real** — You CANNOT reliably update a database and publish a message in two separate operations. One can fail independently of the other.

2. **The outbox pattern makes it atomic** — By writing the event to the same database in the same transaction, you guarantee both happen or neither does.

3. **Polling is simple; CDC is powerful** — Start with polling (every 500ms). Move to Debezium/CDC when you need sub-100ms latency or high throughput.

4. **Consumers must be idempotent** — The outbox relay may deliver events more than once (at-least-once delivery). Consumers must handle duplicates gracefully.

5. **Ordering is guaranteed per aggregate** — Use the aggregate ID as the Kafka partition key to ensure events for the same entity are processed in order.

6. **Clean up the outbox table** — Published events serve no purpose. Delete or partition them to prevent table bloat.

7. **This is the foundation of reliable event-driven architectures** — Nearly every production microservices system uses this pattern (or should be).

---

## What's Next?

Next, we'll explore **Sidecar, Ambassador, and Adapter Patterns** — infrastructure-level patterns for handling cross-cutting concerns like logging, monitoring, retries, and protocol translation without polluting your application code.

See: [06-sidecar-ambassador-adapter.md](./06-sidecar-ambassador-adapter.md)
