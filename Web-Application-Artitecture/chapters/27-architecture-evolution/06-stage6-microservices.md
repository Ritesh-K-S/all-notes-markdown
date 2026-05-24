# Stage 6: Microservices + Message Queues — 1M to 10M Users

> **What you'll learn**: How to break a monolithic application into independent services that can be developed, deployed, and scaled independently, connected by message queues for reliable async communication.

---

## Real-Life Analogy

Your chai business has grown into a **food court empire**. You now sell chai, biryani, pizza, desserts, and fresh juice. Running everything as one kitchen is a nightmare:
- A pizza oven fire shuts down chai too
- The biryani chef's vacation delays juice orders
- You can't scale pizza without also scaling everything else
- 50 cooks in one kitchen keep bumping into each other

**Solution**: You split into **independent restaurants** in the food court. Each has its own kitchen, staff, inventory, and menu. They communicate via a **shared order board** (message queue): "Table 7 ordered chai + pizza" → the chai shop makes chai, the pizza shop makes pizza, independently.

If the pizza oven breaks, chai is still being served. You can hire 5 more pizza cooks without touching the chai team.

---

## Core Concept Explained Step-by-Step

### From Monolith to Microservices

```
BEFORE: Monolith (one big application)
┌──────────────────────────────────────────────────────────────┐
│                      ONE APPLICATION                          │
│                                                              │
│  ┌──────┐ ┌──────────┐ ┌─────────┐ ┌────────┐ ┌──────────┐ │
│  │Users │ │  Orders  │ │Payments │ │ Search │ │Notifications│
│  │      │ │          │ │         │ │        │ │          │ │
│  └──────┘ └──────────┘ └─────────┘ └────────┘ └──────────┘ │
│                                                              │
│  ONE database │ ONE deploy │ ONE team │ ONE codebase         │
└──────────────────────────────────────────────────────────────┘

AFTER: Microservices (independent services)
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │  Order   │  │ Payment  │  │  Search  │  │Notification│
│ Service  │  │ Service  │  │ Service  │  │ Service  │  │ Service  │
│          │  │          │  │          │  │          │  │          │
│ Own DB   │  │ Own DB   │  │ Own DB   │  │  Own DB  │  │ Own DB   │
│ Own team │  │ Own team │  │ Own team │  │ Own team │  │ Own team │
│ Own deploy│ │ Own deploy│ │ Own deploy│ │ Own deploy│ │ Own deploy│
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │              │              │
     └──────────────┴──────────────┴──────────────┴──────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   MESSAGE QUEUE     │
                    │   (Kafka/RabbitMQ)  │
                    │                    │
                    │ Async communication │
                    │ between services   │
                    └────────────────────┘
```

### How Services Communicate

```
TWO WAYS SERVICES TALK TO EACH OTHER:

1. SYNCHRONOUS (HTTP/gRPC) — "I need an answer NOW"
┌────────────┐   HTTP/gRPC    ┌────────────┐
│   Order    │ ──────────────▶│  Payment   │
│  Service   │◀───── response ─│  Service   │
└────────────┘                 └────────────┘
Use when: Result needed immediately (e.g., "Is payment approved?")

2. ASYNCHRONOUS (Message Queue) — "Do this when you can"
┌────────────┐    publish     ┌────────────┐    consume    ┌────────────┐
│   Order    │ ──────────────▶│   KAFKA    │──────────────▶│Notification│
│  Service   │                │   QUEUE    │               │  Service   │
└────────────┘                └────────────┘               └────────────┘
Use when: Result not needed immediately (e.g., "Send confirmation email")
```

### The Full Architecture at Stage 6

```
                              Internet
                                 │
                                 ▼
                        ┌─────────────────┐
                        │   API Gateway   │
                        │  (Kong/Nginx)   │
                        │                 │
                        │ Routes requests │
                        │ to services     │
                        └────────┬────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
          ▼                      ▼                      ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  User Service    │  │  Order Service   │  │  Search Service  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │ 3 instances│  │  │  │ 5 instances│  │  │  │ 2 instances│  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │ PostgreSQL │  │  │  │ PostgreSQL │  │  │  │Elasticsearch│ │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
└────────┬─────────┘  └────────┬─────────┘  └──────────────────┘
         │                     │
         │    ┌────────────────┴────────────────┐
         │    │                                 │
         ▼    ▼                                 ▼
┌──────────────────┐                 ┌──────────────────┐
│   Kafka/RabbitMQ │                 │ Payment Service  │
│   Message Queue  │                 │  ┌────────────┐  │
│                  │────────────────▶│  │ 3 instances│  │
│ Topics:          │                 │  └────────────┘  │
│ • order.created  │                 │  ┌────────────┐  │
│ • payment.done   │◀────────────────│  │ PostgreSQL │  │
│ • user.registered│                 │  └────────────┘  │
└─────────┬────────┘                 └──────────────────┘
          │
          ▼
┌──────────────────┐
│Notification Svc  │
│  (Email/SMS/Push)│
│  ┌────────────┐  │
│  │ 2 instances│  │
│  └────────────┘  │
└──────────────────┘
```

### Why Message Queues Are Essential

```
WITHOUT message queue (direct calls):
┌────────┐      ┌─────────┐      ┌──────────────┐
│ Order  │─────▶│ Payment │─────▶│ Notification │
│Service │      │ Service │      │   Service    │
└────────┘      └─────────┘      └──────────────┘

Problem: If Notification Service is DOWN:
┌────────┐      ┌─────────┐      ┌──────────────┐
│ Order  │─────▶│ Payment │─────▶│ Notification │ ✗ DOWN!
│Service │      │ Service │      │   Service    │
└────────┘      └─────────┘      └──────────────┘
                    │
                    └── Fails! Payment hangs! Order fails!
                        User can't place order because email service is down! 😱

WITH message queue:
┌────────┐      ┌─────────┐      ┌───────┐      ┌──────────────┐
│ Order  │─────▶│ Payment │─────▶│ QUEUE │─────▶│ Notification │ ✗ DOWN
│Service │      │ Service │      │       │      │   Service    │
└────────┘      └─────────┘      └───────┘      └──────────────┘
                    │                  │
                    └── Succeeds! ✓    └── Messages WAIT in queue
                                           When service recovers,
                                           all emails get sent! ✓
```

### The Order Flow (Real Example)

```
User clicks "Place Order":

1. API Gateway → Order Service (HTTP)
   ┌─────────────────────────────────────────────┐
   │ Order Service:                               │
   │   - Validates order                          │
   │   - Saves to orders DB                       │
   │   - Publishes: "order.created" to Kafka      │
   └─────────────────────────────────────────────┘
                         │
                    Kafka Topic: "order.created"
                    Message: {orderId: 123, userId: 42, items: [...], total: 999}
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌───────────┐  ┌───────────┐  ┌────────────┐
   │  Payment  │  │ Inventory │  │Notification│
   │  Service  │  │  Service  │  │  Service   │
   │           │  │           │  │            │
   │ Charges   │  │ Reserves  │  │ Sends      │
   │ card      │  │ stock     │  │ "Order     │
   │           │  │           │  │  confirmed"│
   └─────┬─────┘  └───────────┘  │  email     │
         │                        └────────────┘
         │
    Kafka Topic: "payment.completed"
    Message: {orderId: 123, status: "success", txnId: "abc"}
         │
         ▼
   ┌───────────┐
   │  Order    │
   │  Service  │
   │           │
   │ Updates   │
   │ status to │
   │ "paid"    │
   └───────────┘
```

---

## How It Works Internally

### Service Discovery

How does Service A find Service B?

```
Option 1: Service Registry (Consul, Eureka)
┌────────────┐   "Where is Payment Service?"   ┌────────────────┐
│   Order    │ ────────────────────────────────▶│    Consul      │
│  Service   │                                  │  (Registry)    │
│            │◀─── "10.0.1.5:8080, 10.0.1.6:8080" ─│            │
│            │                                  │  Knows ALL     │
│            │──── HTTP to 10.0.1.5:8080 ──────▶│  services      │
└────────────┘                                  └────────────────┘

Option 2: DNS-based (Kubernetes Services)
┌────────────┐   http://payment-service:8080    ┌────────────────┐
│   Order    │ ────────────────────────────────▶│  Kubernetes    │
│  Service   │                                  │  DNS resolves  │
│            │   (K8s resolves to pod IP)        │  to healthy pod│
└────────────┘                                  └────────────────┘
```

### Message Queue Internals (Kafka)

```
Kafka Architecture:
┌──────────────────────────────────────────────────────────────────┐
│                         KAFKA CLUSTER                             │
│                                                                  │
│  Topic: "order.created"                                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Partition 0: [msg1] [msg4] [msg7] [msg10] ...              │ │
│  │  Partition 1: [msg2] [msg5] [msg8] [msg11] ...              │ │
│  │  Partition 2: [msg3] [msg6] [msg9] [msg12] ...              │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Producers (write):        Consumers (read):                     │
│  - Order Service           - Payment Service (consumer group 1)  │
│                            - Inventory Service (consumer group 2) │
│                            - Notification Svc (consumer group 3)  │
│                                                                  │
│  Each consumer group reads ALL messages independently            │
│  Within a group, partitions are split among instances            │
└──────────────────────────────────────────────────────────────────┘

Why partitions?
- Partition 0 → Payment Instance 1
- Partition 1 → Payment Instance 2  
- Partition 2 → Payment Instance 3
= Parallel processing! 3x throughput!
```

### Database per Service

```
┌──────────────────────────────────────────────────────────────────────┐
│ RULE: Each service owns its own database. NO shared databases!       │
│                                                                      │
│ ✗ WRONG (shared database):                                           │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐                            │
│ │  User    │  │  Order   │  │ Payment  │                            │
│ │ Service  │  │ Service  │  │ Service  │                            │
│ └────┬─────┘  └────┬─────┘  └────┬─────┘                            │
│      │              │              │                                  │
│      └──────────────┼──────────────┘                                  │
│                     ▼                                                 │
│           ┌──────────────────┐                                        │
│           │ ONE BIG DATABASE │ ← Coupling! Can't scale independently │
│           └──────────────────┘                                        │
│                                                                      │
│ ✓ CORRECT (database per service):                                    │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐                            │
│ │  User    │  │  Order   │  │ Payment  │                            │
│ │ Service  │  │ Service  │  │ Service  │                            │
│ └────┬─────┘  └────┬─────┘  └────┬─────┘                            │
│      │              │              │                                  │
│      ▼              ▼              ▼                                  │
│ ┌─────────┐  ┌──────────┐  ┌──────────┐                             │
│ │ Users DB│  │ Orders DB│  │Payments DB│                             │
│ │(Postgres)│ │ (Postgres)│ │ (Postgres)│                             │
│ └─────────┘  └──────────┘  └──────────┘                             │
│                                                                      │
│ Each service can:                                                    │
│ - Use a DIFFERENT database type (Postgres, MongoDB, Redis)           │
│ - Scale its DB independently                                        │
│ - Change schema without affecting others                            │
└──────────────────────────────────────────────────────────────────────┘
```

### Saga Pattern: Distributed Transactions

```
Problem: "Place Order" touches multiple services. What if payment fails AFTER
         inventory was already reserved?

Solution: SAGA — a sequence of local transactions with compensating actions

Happy Path:
┌────────┐    ┌───────────┐    ┌─────────┐    ┌──────────┐
│ Create │───▶│  Reserve  │───▶│ Charge  │───▶│  Confirm │
│ Order  │    │ Inventory │    │ Payment │    │  Order   │
└────────┘    └───────────┘    └─────────┘    └──────────┘

Failure (payment fails):
┌────────┐    ┌───────────┐    ┌─────────┐
│ Create │───▶│  Reserve  │───▶│ Charge  │ ✗ FAILED!
│ Order  │    │ Inventory │    │ Payment │
└────────┘    └───────────┘    └─────────┘
                                     │
                              COMPENSATE! (undo)
                                     │
              ┌───────────┐          │
              │  Release  │◀─────────┘
              │ Inventory │
              └───────────┘
                    │
              ┌─────▼────┐
              │  Cancel  │
              │  Order   │
              └──────────┘
```

---

## Code Examples

### Python: Microservice with Kafka Producer/Consumer

```python
# order_service.py — Order Service that publishes events
from flask import Flask, jsonify, request
from kafka import KafkaProducer
import json
import uuid

app = Flask(__name__)

# Kafka producer: publish events when orders are created
producer = KafkaProducer(
    bootstrap_servers=["kafka1:9092", "kafka2:9092", "kafka3:9092"],
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
    acks="all",              # Wait for all replicas to confirm
    retries=3                # Retry on failure
)

@app.route("/api/orders", methods=["POST"])
def create_order():
    data = request.get_json()
    order_id = str(uuid.uuid4())

    # Step 1: Save order to this service's database
    order = {
        "order_id": order_id,
        "user_id": data["user_id"],
        "items": data["items"],
        "total": data["total"],
        "status": "pending"
    }
    # save_to_db(order)  — saves to Order Service's own database

    # Step 2: Publish event (other services react asynchronously)
    producer.send(
        "order.created",          # Kafka topic
        key=order_id.encode(),    # Partition key (same order → same partition)
        value={
            "event": "order.created",
            "order_id": order_id,
            "user_id": data["user_id"],
            "items": data["items"],
            "total": data["total"],
            "timestamp": "2024-01-15T10:30:00Z"
        }
    )
    producer.flush()  # Ensure message is sent

    return jsonify({"order_id": order_id, "status": "pending"}), 201


# payment_consumer.py — Payment Service consumes order events
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    "order.created",                    # Subscribe to this topic
    bootstrap_servers=["kafka1:9092"],
    group_id="payment-service",         # Consumer group
    value_deserializer=lambda m: json.loads(m.decode("utf-8")),
    auto_offset_reset="earliest"
)

def process_payment(order_event):
    """Process payment for a new order"""
    order_id = order_event["order_id"]
    total = order_event["total"]

    # Charge the customer (call payment gateway)
    # payment_result = charge_card(user_id, total)

    # Publish result
    producer.send("payment.completed", value={
        "event": "payment.completed",
        "order_id": order_id,
        "status": "success",
        "transaction_id": "txn_abc123"
    })

# Main loop: continuously consume and process
for message in consumer:
    process_payment(message.value)
```

### Java: Spring Boot Microservice with Kafka

```java
// OrderService.java — Publishes events to Kafka
@Service
public class OrderService {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Autowired
    private OrderRepository orderRepository;

    public Order createOrder(CreateOrderRequest request) {
        // Step 1: Save to this service's database
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setItems(request.getItems());
        order.setTotal(request.getTotal());
        order.setStatus("PENDING");
        order = orderRepository.save(order);

        // Step 2: Publish event for other services
        OrderEvent event = new OrderEvent(
            "order.created",
            order.getId(),
            order.getUserId(),
            order.getTotal()
        );
        kafkaTemplate.send("order.created", order.getId(), event);

        return order;
    }
}

// PaymentEventListener.java — Consumes events from Kafka
@Service
public class PaymentEventListener {

    @Autowired
    private PaymentGateway paymentGateway;

    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;

    @KafkaListener(topics = "order.created", groupId = "payment-service")
    public void handleOrderCreated(OrderEvent event) {
        // Process payment asynchronously
        PaymentResult result = paymentGateway.charge(
            event.getUserId(), event.getTotal()
        );

        // Publish payment result
        PaymentEvent paymentEvent = new PaymentEvent(
            event.getOrderId(),
            result.isSuccess() ? "SUCCESS" : "FAILED",
            result.getTransactionId()
        );
        kafkaTemplate.send("payment.completed",
            event.getOrderId(), paymentEvent);
    }
}

// application.yml
// spring:
//   kafka:
//     bootstrap-servers: kafka1:9092,kafka2:9092,kafka3:9092
//     consumer:
//       group-id: payment-service
//       auto-offset-reset: earliest
//     producer:
//       acks: all
//       retries: 3
```

---

## Infrastructure Example

### Kubernetes Deployment for Microservices

```yaml
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 5       # 5 instances of Order Service
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myapp/order-service:v2.1.0
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: url
        - name: KAFKA_BROKERS
          value: "kafka-0.kafka:9092,kafka-1.kafka:9092,kafka-2.kafka:9092"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 8080
    targetPort: 8080
```

### Kafka Cluster (Docker Compose for Development)

```yaml
# docker-compose.yml — Kafka + Zookeeper
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka1:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:9092
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3

  kafka2:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:9092

  kafka3:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:9092
```

---

## Real-World Example

### Netflix: 1000+ Microservices

Netflix's architecture:
- **1000+ microservices** running in AWS
- Each service owned by a small team (2-5 people)
- Services communicate via REST + Kafka events
- Key services: User Service, Recommendation Engine, Video Streaming, Billing
- Each can scale independently: Streaming needs 10x servers, Billing needs 2

### Uber: Event-Driven Order Processing

Uber's ride flow uses Kafka heavily:
1. **Ride Request** → published to Kafka
2. **Matching Service** consumes → finds nearby drivers
3. **Driver Accept** → published to Kafka
4. **Pricing Service** consumes → calculates fare
5. **ETA Service** consumes → estimates arrival
6. **Notification Service** consumes → sends push notification to rider

All happening **in parallel**, each service scaling independently.

### Amazon: Hundreds of Services per Page

When you load a product page on Amazon:
- **Product Service**: title, description
- **Price Service**: current price, discounts
- **Review Service**: ratings, reviews
- **Recommendation Service**: "Customers also bought..."
- **Inventory Service**: "Only 3 left in stock"
- **Shipping Service**: delivery estimate

Each is a separate microservice, each can fail independently without killing the whole page.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Breaking into microservices too early** | Adds complexity before you understand domain boundaries | Start monolith, split later when boundaries are clear |
| **Shared database between services** | Tight coupling; can't deploy independently | Each service owns its data (database per service) |
| **Not handling message failures** | Lost messages = lost orders | Dead letter queue (DLQ) + retry with backoff |
| **Synchronous calls everywhere** | One slow service brings everything down | Use async messaging for non-critical paths |
| **No circuit breakers** | Cascading failures across services | Implement circuit breakers (Resilience4j, Hystrix) |
| **Too fine-grained services** | "Nano-services" with too much network overhead | Keep services aligned to business domains |
| **Not investing in observability** | Can't debug across 20 services | Distributed tracing (Jaeger), centralized logging (ELK) |
| **Ignoring data consistency** | Eventual consistency bugs | Design for it: idempotent consumers, saga pattern |

---

## When to Use / When NOT to Use

### ✅ Move to Microservices When:
- Your monolith is too large for one team to manage
- Different features need different scaling (search vs checkout)
- Teams are blocked by each other during deploys
- You need technology diversity (ML service in Python, API in Java)
- Failure in one feature should NOT bring down everything
- You're at 1M+ users with 20+ engineers

### ❌ Don't Use Microservices When:
- You have a small team (< 10 engineers)
- You're still figuring out your product/domain
- You don't have DevOps maturity (CI/CD, monitoring, containers)
- A well-structured monolith is handling the load fine
- You can't invest in infrastructure (Kubernetes, service mesh, observability)

> **Martin Fowler's Rule**: "Don't start with microservices. Start with a monolith, keep it modular, and split when you have a GOOD REASON."

---

## Key Takeaways

1. **Microservices = independently deployable services, each owning its data** — NOT just "smaller code"
2. **Message queues (Kafka/RabbitMQ) decouple services** — failures don't cascade, processing is async
3. **Each service gets its own database** — no shared state, no coupling
4. **This adds significant complexity** — distributed tracing, eventual consistency, network failures
5. **You need organizational maturity** — CI/CD, monitoring, on-call, containerization
6. **Start monolith, extract services when needed** — premature microservices is a top mistake
7. **Event-driven communication is the backbone** — services publish events, others react independently

---

## What's Next?

With microservices, you can scale individual services independently. But what happens when 10M users become 100M, spread across multiple countries? When a single database (even with replicas) can't hold all the data? When latency matters so much that you need servers on every continent?

That's when you need **Sharding + Multi-Region Deployment** — Stage 7.

Next: [07-stage7-sharding-multi-region.md](./07-stage7-sharding-multi-region.md)
