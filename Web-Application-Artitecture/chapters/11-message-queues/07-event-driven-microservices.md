# Event-Driven Microservices with Kafka/RabbitMQ

> **What you'll learn**: How to architect a complete event-driven microservices system using message brokers — including event design, choreography vs orchestration, saga patterns, schema evolution, and production patterns used by Netflix, Uber, and Amazon.

---

## Real-Life Analogy

Think of a **busy hospital emergency room**.

In a **command-driven** (synchronous) hospital, the head doctor would personally coordinate everything:
- "Nurse, take blood pressure!" → waits → "Lab, run these tests!" → waits → "Pharmacy, prepare medication!" → waits
- If the lab is busy, EVERYONE waits. If the doctor is overwhelmed, the system collapses.

In an **event-driven** hospital, everyone reacts to events independently:
- Patient arrives → "PatientArrived" event announced over PA system
- Triage nurse hears it → starts assessment
- Registration desk hears it → starts paperwork
- Lab tech hears it → prepares equipment
- Each department works **independently** at their own pace. No one waits. No central coordinator bottleneck.

If the lab is slow, it doesn't block the nurse. If registration crashes, patients are still treated. This is event-driven architecture — **autonomous services reacting to events**.

---

## Core Concept Explained Step-by-Step

### Step 1: What Are Event-Driven Microservices?

**Microservices** = Small, independent services that each own one business capability.

**Event-Driven** = Services communicate by publishing and consuming events, not by calling each other directly.

```
TRADITIONAL MICROSERVICES (Request-Driven):

┌─────────┐  HTTP  ┌─────────┐  HTTP  ┌─────────┐  HTTP  ┌─────────┐
│  Order  │──────▶│ Payment │──────▶│Inventory│──────▶│  Email  │
│ Service │◀──────│ Service │◀──────│ Service │◀──────│ Service │
└─────────┘       └─────────┘       └─────────┘       └─────────┘

Problems:
- Tight coupling (Order Service KNOWS about all downstream services)
- Cascading failures (Payment down → everything down)
- Hard to add new services (must modify Order Service)


EVENT-DRIVEN MICROSERVICES:

┌─────────┐                                    ┌─────────┐
│  Order  │───publish───▶ "OrderCreated" ◀───subscribe──│ Payment │
│ Service │              event                  │ Service │
└─────────┘                                    └─────────┘
                              │
                              │              ┌──────────┐
                              └──subscribe───│Inventory │
                              │              │ Service  │
                              │              └──────────┘
                              │              ┌──────────┐
                              └──subscribe───│  Email   │
                                             │ Service  │
                                             └──────────┘

Benefits:
- Loose coupling (Order Service doesn't know who listens)
- Resilient (if Email is down, others continue)
- Easy to extend (add Analytics service — no changes to Order Service!)
```

### Step 2: Events vs Commands

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  EVENT (Something happened)          COMMAND (Do something)         │
│  ═════════════════════════           ═══════════════════════        │
│                                                                     │
│  "OrderCreated"                      "CreateOrder"                  │
│  "PaymentProcessed"                  "ProcessPayment"               │
│  "ItemShipped"                       "ShipItem"                     │
│                                                                     │
│  • Past tense                        • Imperative                   │
│  • States a FACT                     • Requests an ACTION           │
│  • Publisher doesn't care            • Sender expects a result      │
│    who processes it                                                 │
│  • Can have 0 or many               • Has exactly 1 handler        │
│    subscribers                                                      │
│                                                                     │
│  EVENT-DRIVEN = Services react       COMMAND-DRIVEN = Services      │
│  to what happened                    are told what to do            │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### Step 3: Choreography vs Orchestration

Two ways to coordinate multi-service workflows:

**Choreography** (Decentralized — each service knows what to do):

```
CHOREOGRAPHY: Each service reacts to events and emits new events

┌─────────┐  OrderCreated  ┌─────────┐  PaymentDone  ┌──────────┐
│  Order  │───────────────▶│ Payment │───────────────▶│Inventory │
│ Service │                │ Service │                │ Service  │
└─────────┘                └─────────┘                └────┬─────┘
                                                           │
                                                    ItemReserved
                                                           │
                                                           ▼
                                                    ┌──────────┐
                                                    │ Shipping │
                                                    │ Service  │
                                                    └──────────┘

No central coordinator! Each service:
1. Listens for relevant events
2. Does its work
3. Publishes outcome events
4. Other services react to those events

✅ Pros: Decoupled, scalable, no single point of failure
❌ Cons: Hard to see full workflow, complex error handling
```

**Orchestration** (Centralized — one service coordinates):

```
ORCHESTRATION: A central orchestrator tells each service what to do

                    ┌───────────────────┐
                    │   Order Saga      │
                    │   Orchestrator    │
                    └───────┬───────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
      ┌──────────┐   ┌──────────┐   ┌──────────┐
      │ Payment  │   │Inventory │   │ Shipping │
      │ Service  │   │ Service  │   │ Service  │
      └──────────┘   └──────────┘   └──────────┘

Orchestrator sends commands:
1. "ProcessPayment" → Payment Service
2. Wait for result
3. "ReserveInventory" → Inventory Service
4. Wait for result
5. "ScheduleShipment" → Shipping Service

✅ Pros: Clear workflow visibility, easier error handling
❌ Cons: Central point of failure, tighter coupling
```

### Step 4: The Saga Pattern — Managing Distributed Transactions

In microservices, you can't use a single database transaction across services. The **Saga pattern** manages multi-step workflows with compensation:

```
SAGA: Order Fulfillment (Choreography-based)

HAPPY PATH:
OrderCreated → PaymentCharged → InventoryReserved → OrderConfirmed ✓

FAILURE PATH (Payment succeeds, but Inventory fails):
OrderCreated → PaymentCharged → InventoryFailed!
                     │
                     ▼
              COMPENSATION:
              PaymentRefunded ← (undo the payment!)
              OrderCancelled  ← (mark order as failed)

┌─────────────────────────────────────────────────────────────────────┐
│                     SAGA COMPENSATION FLOW                            │
│                                                                      │
│  Step 1        Step 2         Step 3         Step 4                  │
│  ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│  │ Create  │──▶│  Charge  │──▶│ Reserve  │──▶│  Ship    │         │
│  │ Order   │   │ Payment  │   │Inventory │   │  Order   │         │
│  └─────────┘   └──────────┘   └──────────┘   └──────────┘         │
│                                      │                               │
│                                   FAILS!                             │
│                                      │                               │
│  Compensate:   Compensate:           │                               │
│  ┌─────────┐   ┌──────────┐         │                               │
│  │ Cancel  │◀──│  Refund  │◀────────┘                               │
│  │ Order   │   │ Payment  │                                          │
│  └─────────┘   └──────────┘                                         │
│                                                                      │
│  Each step has a COMPENSATING action that undoes it.                │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 5: Event Design — What Goes in an Event?

```
┌──────────────────────────────────────────────────────────────────┐
│                    EVENT STRUCTURE                                 │
│                                                                    │
│  {                                                                │
│    "event_id": "evt-789abc",           ← Unique (for idempotency) │
│    "event_type": "OrderCreated",       ← What happened            │
│    "timestamp": "2024-01-15T10:30:00Z",← When                     │
│    "version": "1.2",                   ← Schema version           │
│    "source": "order-service",          ← Who published            │
│    "correlation_id": "req-123",        ← Trace across services    │
│    "data": {                           ← Event payload            │
│      "order_id": "ORD-456",                                       │
│      "user_id": "USR-789",                                        │
│      "items": [...],                                               │
│      "total_amount": 149.99                                        │
│    }                                                               │
│  }                                                                │
└──────────────────────────────────────────────────────────────────┘

TWO APPROACHES:

1. FAT EVENT (carry all data):
   + Consumer has everything it needs
   + No need to call back to source
   - Large message size
   - Exposes internal data structure

2. THIN EVENT (minimal data + reference):
   + Small message size
   + Loose coupling
   - Consumer must fetch details separately
   - Additional latency
```

---

## How It Works Internally

### Event Flow Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                COMPLETE EVENT-DRIVEN ARCHITECTURE                          │
│                                                                            │
│  ┌────────────┐                                                           │
│  │   Client   │                                                           │
│  │  (Mobile/  │                                                           │
│  │   Web)     │                                                           │
│  └─────┬──────┘                                                           │
│        │ REST/GraphQL                                                     │
│        ▼                                                                  │
│  ┌────────────┐                                                           │
│  │    API     │                                                           │
│  │  Gateway   │                                                           │
│  └─────┬──────┘                                                           │
│        │                                                                  │
│        ▼                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                      KAFKA CLUSTER                                  │  │
│  │                                                                     │  │
│  │  Topics:                                                           │  │
│  │  ┌────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐ │  │
│  │  │   orders   │ │   payments   │ │  inventory   │ │notifications│ │  │
│  │  └────────────┘ └──────────────┘ └──────────────┘ └────────────┘ │  │
│  │  ┌────────────┐ ┌──────────────┐ ┌──────────────┐               │  │
│  │  │  shipping  │ │  analytics   │ │  orders.dlq  │               │  │
│  │  └────────────┘ └──────────────┘ └──────────────┘               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│        │              │              │              │                     │
│        ▼              ▼              ▼              ▼                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │  Order   │  │ Payment  │  │Inventory │  │  Email   │               │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │               │
│  │          │  │          │  │          │  │          │               │
│  │ Postgres │  │ Postgres │  │  Redis   │  │  (none)  │               │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │
│                                                                          │
│  Each service:                                                           │
│  - Owns its own database (Database per Service pattern)                 │
│  - Publishes events to Kafka                                            │
│  - Subscribes to relevant events from other services                    │
│  - Has its own consumer group                                           │
│  - Is independently deployable and scalable                             │
└──────────────────────────────────────────────────────────────────────────┘
```

### The Outbox Pattern — Reliable Event Publishing

Problem: How do you atomically update your database AND publish an event?

```
PROBLEM — Dual Write (UNSAFE):

Service                    Database              Kafka
   │                          │                    │
   │── save order ──────────▶ │                    │
   │◀── saved ✓ ─────────────│                    │
   │                          │                    │
   │── publish event ─────────────────────────────▶│
   │       ✗ CRASH! ✗                              │
   │                                               │
   │  Order saved in DB, but event NEVER published!
   │  Other services don't know about the order!
   
SOLUTION — Outbox Pattern:

Service                    Database              CDC / Poller       Kafka
   │                          │                      │                │
   │── BEGIN TRANSACTION ────▶│                      │                │
   │   INSERT order           │                      │                │
   │   INSERT into OUTBOX     │  ← event stored     │                │
   │── COMMIT ───────────────▶│    in same TX!       │                │
   │                          │                      │                │
   │                          │── CDC reads ────────▶│                │
   │                          │   outbox table       │── publish ────▶│
   │                          │                      │                │
   
If crash before COMMIT: nothing persists (safe!)
If crash after COMMIT: CDC will eventually publish the event (safe!)
ATOMIC: DB write + event publish are guaranteed together.
```

```
OUTBOX TABLE:

┌────────────────────────────────────────────────────────────────┐
│                      outbox_events                               │
├───────┬────────────────┬──────────────┬──────────┬─────────────┤
│  id   │  event_type    │   payload    │  status  │ created_at  │
├───────┼────────────────┼──────────────┼──────────┼─────────────┤
│  1    │ OrderCreated   │ {order:...}  │ SENT     │ 10:30:01    │
│  2    │ OrderUpdated   │ {order:...}  │ PENDING  │ 10:30:05    │
│  3    │ OrderCancelled │ {order:...}  │ PENDING  │ 10:30:09    │
└───────┴────────────────┴──────────────┴──────────┴─────────────┘

A background process (or CDC tool like Debezium) reads PENDING rows
and publishes them to Kafka, then marks them as SENT.
```

### Schema Evolution — Handling Event Format Changes

```
VERSION 1:                        VERSION 2 (new field added):
{                                 {
  "event_type": "OrderCreated",     "event_type": "OrderCreated",
  "order_id": "ORD-123",           "order_id": "ORD-123",
  "user_id": "USR-456",            "user_id": "USR-456",
  "amount": 99.99                   "amount": 99.99,
}                                   "currency": "USD",        ← NEW!
                                    "discount_code": null      ← NEW!
                                  }

RULES FOR SAFE EVOLUTION:
✅ Add new OPTIONAL fields (backward compatible)
✅ Provide default values for new fields
❌ NEVER remove existing fields
❌ NEVER change field types
❌ NEVER rename fields

Use: Apache Avro + Schema Registry for enforcement
```

---

## Code Examples

### Python — Complete Event-Driven Order Service

```python
# ============ ORDER SERVICE (Producer) ============
from confluent_kafka import Producer
import json
import uuid
from datetime import datetime
from sqlalchemy import create_engine, text

class OrderService:
    """Order service that publishes events to Kafka"""
    
    def __init__(self):
        self.producer = Producer({
            'bootstrap.servers': 'localhost:9092',
            'acks': 'all',
            'enable.idempotence': True,
        })
        self.db = create_engine('postgresql://localhost/orders')
    
    def create_order(self, user_id, items, total):
        """Create order using Outbox Pattern for reliable publishing"""
        order_id = f"ORD-{uuid.uuid4().hex[:8]}"
        event_id = str(uuid.uuid4())
        
        event = {
            "event_id": event_id,
            "event_type": "OrderCreated",
            "timestamp": datetime.utcnow().isoformat(),
            "source": "order-service",
            "data": {
                "order_id": order_id,
                "user_id": user_id,
                "items": items,
                "total_amount": total,
                "status": "PENDING"
            }
        }
        
        # ATOMIC: Save order + outbox event in same transaction
        with self.db.begin() as conn:
            conn.execute(text("""
                INSERT INTO orders (id, user_id, total, status)
                VALUES (:id, :user_id, :total, 'PENDING')
            """), {"id": order_id, "user_id": user_id, "total": total})
            
            conn.execute(text("""
                INSERT INTO outbox_events (id, event_type, payload, status)
                VALUES (:id, :type, :payload, 'PENDING')
            """), {"id": event_id, "type": "OrderCreated", 
                   "payload": json.dumps(event)})
        
        # Publish to Kafka (also done by CDC in production)
        self.producer.produce(
            'orders', key=user_id, value=json.dumps(event).encode()
        )
        self.producer.flush()
        
        return {"order_id": order_id, "status": "created"}


# ============ PAYMENT SERVICE (Consumer + Producer) ============
class PaymentService:
    """Listens for OrderCreated, processes payment, emits PaymentProcessed"""
    
    def __init__(self):
        self.consumer = Consumer({
            'bootstrap.servers': 'localhost:9092',
            'group.id': 'payment-service',
            'enable.auto.commit': False,
        })
        self.producer = Producer({
            'bootstrap.servers': 'localhost:9092',
            'acks': 'all',
        })
        self.consumer.subscribe(['orders'])
    
    def run(self):
        while True:
            msg = self.consumer.poll(1.0)
            if msg is None:
                continue
            
            event = json.loads(msg.value())
            
            if event['event_type'] == 'OrderCreated':
                self.handle_order_created(event)
            
            self.consumer.commit(asynchronous=False)
    
    def handle_order_created(self, event):
        """Process payment and publish result"""
        order = event['data']
        
        try:
            # Charge payment (idempotent using order_id)
            charge_result = self.charge_payment(
                order['user_id'], order['total_amount'], order['order_id']
            )
            
            # Publish success event
            payment_event = {
                "event_id": str(uuid.uuid4()),
                "event_type": "PaymentProcessed",
                "correlation_id": event['event_id'],
                "data": {
                    "order_id": order['order_id'],
                    "payment_id": charge_result['payment_id'],
                    "status": "SUCCESS"
                }
            }
            self.producer.produce(
                'payments', key=order['order_id'],
                value=json.dumps(payment_event).encode()
            )
            
        except PaymentError as e:
            # Publish failure event (triggers compensation)
            failure_event = {
                "event_id": str(uuid.uuid4()),
                "event_type": "PaymentFailed",
                "data": {
                    "order_id": order['order_id'],
                    "reason": str(e)
                }
            }
            self.producer.produce(
                'payments', key=order['order_id'],
                value=json.dumps(failure_event).encode()
            )
        
        self.producer.flush()
```

### Python — Saga Orchestrator

```python
# ============ SAGA ORCHESTRATOR ============
class OrderSagaOrchestrator:
    """Manages the order fulfillment saga with compensation"""
    
    STEPS = [
        {"action": "reserve_inventory", "compensation": "release_inventory"},
        {"action": "process_payment", "compensation": "refund_payment"},
        {"action": "schedule_shipping", "compensation": "cancel_shipping"},
    ]
    
    def __init__(self):
        self.consumer = Consumer({
            'bootstrap.servers': 'localhost:9092',
            'group.id': 'saga-orchestrator',
            'enable.auto.commit': False,
        })
        self.producer = Producer({'bootstrap.servers': 'localhost:9092'})
        self.consumer.subscribe(['saga-replies'])
    
    def execute_saga(self, order):
        """Execute saga steps, compensate on failure"""
        completed_steps = []
        
        for step in self.STEPS:
            # Send command to service
            command = {
                "saga_id": order['order_id'],
                "action": step['action'],
                "data": order
            }
            self.producer.produce(
                f"commands.{step['action']}", 
                value=json.dumps(command).encode()
            )
            self.producer.flush()
            
            # Wait for reply
            reply = self.wait_for_reply(order['order_id'], timeout=30)
            
            if reply['status'] == 'SUCCESS':
                completed_steps.append(step)
            else:
                # FAILURE: Compensate all completed steps (in reverse)
                print(f"Step '{step['action']}' failed! Compensating...")
                self.compensate(completed_steps, order)
                return {"status": "FAILED", "reason": reply.get('error')}
        
        return {"status": "COMPLETED"}
    
    def compensate(self, completed_steps, order):
        """Execute compensation actions in reverse order"""
        for step in reversed(completed_steps):
            compensation = {
                "saga_id": order['order_id'],
                "action": step['compensation'],
                "data": order
            }
            self.producer.produce(
                f"commands.{step['compensation']}",
                value=json.dumps(compensation).encode()
            )
        self.producer.flush()
        print(f"Compensation complete for order {order['order_id']}")
```

### Java — Event-Driven Microservice with Spring Cloud Stream

```java
import org.springframework.cloud.stream.annotation.*;
import org.springframework.cloud.stream.messaging.Processor;
import org.springframework.messaging.handler.annotation.SendTo;

// ============ ORDER SERVICE ============
@Service
public class OrderEventPublisher {
    
    private final StreamBridge streamBridge;
    
    public OrderEventPublisher(StreamBridge streamBridge) {
        this.streamBridge = streamBridge;
    }
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // Save order to DB
        Order order = orderRepository.save(new Order(request));
        
        // Publish event via Spring Cloud Stream
        OrderCreatedEvent event = new OrderCreatedEvent(
            UUID.randomUUID().toString(),
            "OrderCreated",
            Instant.now(),
            order.getId(),
            order.getUserId(),
            order.getTotalAmount()
        );
        
        streamBridge.send("orders-out-0", event);
        return order;
    }
}

// ============ PAYMENT SERVICE (Consumer) ============
@Service
public class PaymentEventProcessor {
    
    @Bean
    public Function<OrderCreatedEvent, PaymentEvent> processPayment() {
        return orderEvent -> {
            try {
                // Process payment
                Payment payment = paymentGateway.charge(
                    orderEvent.getUserId(),
                    orderEvent.getAmount()
                );
                
                return new PaymentProcessedEvent(
                    orderEvent.getOrderId(),
                    payment.getId(),
                    "SUCCESS"
                );
            } catch (PaymentException e) {
                return new PaymentFailedEvent(
                    orderEvent.getOrderId(),
                    e.getMessage()
                );
            }
        };
    }
}

// ============ INVENTORY SERVICE (Consumer) ============
@Service  
public class InventoryEventProcessor {
    
    @Bean
    public Consumer<PaymentProcessedEvent> reserveInventory() {
        return event -> {
            // Only react to successful payments
            Order order = orderClient.getOrder(event.getOrderId());
            
            for (OrderItem item : order.getItems()) {
                inventoryRepository.decreaseStock(
                    item.getProductId(), item.getQuantity()
                );
            }
            
            // Publish inventory reserved event
            streamBridge.send("inventory-out-0", 
                new InventoryReservedEvent(event.getOrderId()));
        };
    }
}
```

### Java — Outbox Pattern with Debezium

```java
// ============ OUTBOX ENTITY ============
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    private String id;
    private String aggregateType;    // e.g., "Order"
    private String aggregateId;      // e.g., order_id
    private String eventType;        // e.g., "OrderCreated"
    
    @Column(columnDefinition = "jsonb")
    private String payload;
    
    private Instant createdAt;
}

// ============ SERVICE using Outbox ============
@Service
public class OrderServiceWithOutbox {
    
    private final OrderRepository orderRepo;
    private final OutboxRepository outboxRepo;
    
    @Transactional  // SAME transaction for both!
    public Order createOrder(CreateOrderRequest req) {
        // 1. Save the order
        Order order = orderRepo.save(new Order(req));
        
        // 2. Write event to outbox (same transaction!)
        OutboxEvent event = new OutboxEvent();
        event.setId(UUID.randomUUID().toString());
        event.setAggregateType("Order");
        event.setAggregateId(order.getId());
        event.setEventType("OrderCreated");
        event.setPayload(toJson(order));
        event.setCreatedAt(Instant.now());
        
        outboxRepo.save(event);
        
        // Debezium (CDC) watches the outbox table and publishes to Kafka
        // automatically. The outbox row is deleted after publishing.
        
        return order;
    }
}
```

---

## Infrastructure Examples

### Complete Event-Driven Setup with Docker Compose

```yaml
version: '3.8'
services:
  # ─── Kafka Infrastructure ───
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,EXTERNAL://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on: [kafka]
    ports: ["8081:8081"]
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:29092

  # ─── CDC (Change Data Capture) for Outbox Pattern ───
  debezium:
    image: debezium/connect:2.4
    depends_on: [kafka, order-db]
    ports: ["8083:8083"]
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: debezium-cdc
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets

  # ─── Databases (one per service) ───
  order-db:
    image: postgres:16
    environment:
      POSTGRES_DB: orders
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]
    command: ["postgres", "-c", "wal_level=logical"]  # Required for Debezium

  payment-db:
    image: postgres:16
    environment:
      POSTGRES_DB: payments
      POSTGRES_PASSWORD: secret
    ports: ["5433:5432"]

  inventory-cache:
    image: redis:7
    ports: ["6379:6379"]

  # ─── Microservices ───
  order-service:
    build: ./order-service
    depends_on: [kafka, order-db]
    environment:
      DATABASE_URL: postgresql://postgres:secret@order-db:5432/orders
      KAFKA_BOOTSTRAP: kafka:29092
    ports: ["8001:8080"]

  payment-service:
    build: ./payment-service
    depends_on: [kafka, payment-db]
    environment:
      DATABASE_URL: postgresql://postgres:secret@payment-db:5432/payments
      KAFKA_BOOTSTRAP: kafka:29092
    ports: ["8002:8080"]

  inventory-service:
    build: ./inventory-service
    depends_on: [kafka, inventory-cache]
    environment:
      REDIS_URL: redis://inventory-cache:6379
      KAFKA_BOOTSTRAP: kafka:29092
    ports: ["8003:8080"]

  notification-service:
    build: ./notification-service
    depends_on: [kafka]
    environment:
      KAFKA_BOOTSTRAP: kafka:29092
    deploy:
      replicas: 2  # Scale consumers independently

  # ─── Monitoring ───
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8080:8080"]
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
```

### Debezium Connector Configuration (Outbox Pattern)

```json
{
  "name": "order-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "order-db",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "secret",
    "database.dbname": "orders",
    "table.include.list": "public.outbox_events",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.topic.replacement": "${routedByValue}",
    "transforms.outbox.table.expand.json.payload": "true"
  }
}
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:v1.2.0
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka-cluster-kafka-bootstrap:9092"
        - name: KAFKA_CONSUMER_GROUP
          value: "order-service"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
        selector:
          matchLabels:
            topic: orders
            group: order-service
      target:
        type: AverageValue
        averageValue: "1000"  # Scale up if lag > 1000 per pod
```

---

## Real-World Example

### Uber — Event-Driven Trip Lifecycle

```
UBER'S EVENT-DRIVEN ARCHITECTURE:

Rider opens app → Requests ride
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         KAFKA                                          │
│                                                                        │
│  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────┐  │
│  │  ride_requests   │   │  driver_updates  │   │  trip_events     │  │
│  └─────────────────┘   └──────────────────┘   └──────────────────┘  │
│  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────┐  │
│  │  pricing_events │   │  payment_events  │   │  fraud_signals   │  │
│  └─────────────────┘   └──────────────────┘   └──────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
         │            │            │            │            │
         ▼            ▼            ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Matching │ │ Pricing  │ │ Payment  │ │   ETA    │ │  Fraud   │
   │ Service  │ │ Service  │ │ Service  │ │ Service  │ │Detection │
   │          │ │          │ │          │ │          │ │ Service  │
   │ Consumes:│ │ Consumes:│ │ Consumes:│ │ Consumes:│ │ Consumes:│
   │ ride_req │ │ ride_req │ │ trip_evts│ │ driver_  │ │ all      │
   │ driver_  │ │ demand   │ │          │ │ updates  │ │ events   │
   │ updates  │ │ signals  │ │          │ │          │ │          │
   └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘

EVENT FLOW for a single ride:
1. RideRequested → Matching finds driver
2. DriverAssigned → ETA calculates arrival time
3. DriverArrived → Trip timer starts
4. TripStarted → Pricing calculates fare in real-time
5. TripCompleted → Payment charges rider, pays driver
6. FraudCheck → Analyzes trip for anomalies

Each service: independent deployment, own database, own scaling.
Uber processes 1+ TRILLION Kafka messages per day.
```

### Netflix — Content Pipeline Events

Netflix's content processing is fully event-driven:
- **VideoUploaded** → triggers transcoding, thumbnail generation, quality checks
- **TranscodingCompleted** → triggers CDN distribution, metadata extraction
- **ContentPublished** → triggers recommendation engine update, search indexing
- **UserWatched** → triggers personalization, "continue watching", billing

Each step publishes events that trigger the next, with no central orchestrator.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Shared database between services | Tight coupling, defeats the purpose of microservices | Database per service + events for synchronization |
| No event schema management | Breaking changes crash consumers | Use Schema Registry (Avro/Protobuf) + compatibility rules |
| Mixing events and commands | Unclear contracts, confusing flow | Events = past tense facts. Commands = imperatives to specific services |
| No correlation ID in events | Impossible to trace a request across services | Add correlation_id to every event, propagate through chain |
| Dual writes (DB + Kafka separately) | Data inconsistency if one fails | Use Outbox Pattern or CDC (Debezium) |
| Not handling out-of-order events | Corrupted state (e.g., "ShipOrder" before "PaymentReceived") | Use event versioning, timestamps, or saga state machines |
| Over-engineering with events for simple CRUD | Massive complexity for no benefit | If 2-3 services and simple flow, use REST. Don't event-drive everything |
| No dead letter queue | Failed events disappear, silent data loss | ALWAYS configure DLQ + alerting |
| Giant monolithic events | Tight coupling, consumers parse unused fields | Small focused events. One event per state change |

---

## When to Use / When NOT to Use

### ✅ Use Event-Driven Microservices When:

- **Multiple services need to react** to the same business event
- **Services should be independently deployable** and scalable
- **High throughput** — events are processed asynchronously at each service's pace
- **Decoupling is critical** — teams own different services, release independently
- **Complex workflows** with many steps (order fulfillment, user onboarding)
- **Event sourcing** — you need a complete history of everything that happened
- **Real-time data pipelines** — streaming analytics, ML feature stores

### ❌ Don't Use Event-Driven Microservices When:

- **Simple CRUD application** — 1-3 services with straightforward request/response
- **Strong consistency required everywhere** — eventual consistency is unacceptable
- **Small team** (< 5 engineers) — operational overhead of Kafka + microservices is significant
- **Low throughput system** — 100 requests/day doesn't justify the complexity
- **Tight deadlines** — building event infrastructure takes time
- **Synchronous workflows only** — if every step MUST complete before the next starts

### Architecture Decision Flowchart:

```
Start
  │
  ├── Is your system > 5 services?
  │     │
  │     ├── YES → Do multiple services need the same data/events?
  │     │           │
  │     │           ├── YES → EVENT-DRIVEN MICROSERVICES ✓
  │     │           │
  │     │           └── NO → Request-driven microservices (REST/gRPC) may suffice
  │     │
  │     └── NO → Monolith or modular monolith is probably fine
  │
  ├── Do you need to process > 10K events/second?
  │     │
  │     └── YES → EVENT-DRIVEN with Kafka ✓
  │
  └── Do you need eventual consistency (not real-time sync)?
        │
        └── YES → EVENT-DRIVEN ✓
```

---

## Key Takeaways

- **Event-driven microservices** communicate through events (facts about what happened), not direct API calls. This provides loose coupling, resilience, and scalability.
- **Choreography** (decentralized, services react independently) is simpler but harder to debug. **Orchestration** (central coordinator) is easier to understand but creates a single point of coordination.
- The **Saga Pattern** handles distributed transactions through a sequence of steps with compensating actions for rollback.
- The **Outbox Pattern** (or CDC with Debezium) solves the dual-write problem — ensuring database updates and event publishing are atomic.
- **Event design** matters: use past-tense names, include correlation IDs, version your schemas, and choose between fat and thin events.
- **Schema evolution** is critical — use a Schema Registry to prevent breaking changes from crashing consumers.
- Start with events only where they provide clear value. Not every service interaction needs to be event-driven.

---

## What's Next?

Congratulations! You've completed **Part 11: Message Queues & Event Streaming**. You now understand synchronous vs asynchronous communication, message queues (RabbitMQ, SQS), Pub/Sub patterns, Apache Kafka, Dead Letter Queues, delivery guarantees, and event-driven microservices architecture.

These patterns form the backbone of how companies like Netflix, Uber, Amazon, and LinkedIn build systems that handle millions of events per second. In the next part of this guide, we'll explore how to make these systems **observable** — monitoring, logging, distributed tracing, and alerting — so you can understand what's happening inside your event-driven architecture in production.
