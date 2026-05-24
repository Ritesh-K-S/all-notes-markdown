# Microservices Architecture — Small, Independent & Powerful

> **What you'll learn**: How microservices work, why companies like Netflix and Uber use them, the trade-offs involved, and how to design systems where each service does one thing well.

---

## Real-Life Analogy

Imagine transforming our single restaurant (monolith) into a **food court**:

- **Pizza counter** — only makes pizza (Pizza Service)
- **Burger counter** — only makes burgers (Burger Service)
- **Drinks counter** — only handles beverages (Drinks Service)
- **Billing counter** — only handles payments (Payment Service)
- **Each counter has its own staff, kitchen, and cash register**

If the pizza oven breaks, **burgers and drinks still work**. If the burger counter gets overwhelmed during lunch, you can **add more burger staff** without touching other counters. If you want to replace the drink menu, you only change the drinks counter.

That's microservices — **small, independent services** that each do one thing, deploy independently, and can fail without taking down the entire system.

---

## Core Concept Explained Step-by-Step

### What are Microservices?

**Microservices architecture** decomposes an application into a collection of **small, autonomous services** that:
1. Each service **does one thing** (Single Responsibility Principle)
2. Each service **owns its own data** (no shared database)
3. Services communicate over the **network** (HTTP, gRPC, events)
4. Each service is **independently deployable** (different release cycles)
5. Each service can use **different technologies** (polyglot)

```
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY                                   │
└─────┬──────────┬──────────┬──────────┬──────────┬───────────────┘
      │          │          │          │          │
      ▼          ▼          ▼          ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐
│  User   │ │ Product │ │  Order  │ │ Payment │ │Notification │
│ Service │ │ Service │ │ Service │ │ Service │ │   Service   │
│         │ │         │ │         │ │         │ │             │
│ Python  │ │  Java   │ │  Go     │ │  Java   │ │   Node.js   │
│ + Mongo │ │ + Postgres│ │ + Postgres│ │ + Redis │ │  + SQS     │
└─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────────┘
     │           │           │           │             │
     ▼           ▼           ▼           ▼             ▼
 ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐     ┌───────┐
 │MongoDB│  │Postgres│  │Postgres│  │ Redis │     │  SQS  │
 └───────┘  └───────┘  └───────┘  └───────┘     └───────┘
```

### The Defining Characteristics

| Principle | What It Means |
|---|---|
| **Single Responsibility** | Each service handles one business capability |
| **Autonomy** | Teams own services end-to-end (dev + deploy + monitor) |
| **Decentralized Data** | Each service has its own database |
| **Smart Endpoints, Dumb Pipes** | Logic lives in services, not in the middleware |
| **Design for Failure** | Assume any service can fail at any time |
| **Independent Deployment** | Deploy one service without redeploying others |
| **Polyglot** | Use the best tech for each service |

---

## How It Works Internally

### Service-to-Service Communication

```
SYNCHRONOUS (Request-Response):

┌────────────┐  HTTP/gRPC   ┌────────────┐
│   Order    │─────────────▶│  Product   │
│  Service   │◀─────────────│  Service   │
└────────────┘   Response    └────────────┘

ASYNCHRONOUS (Event-Driven):

┌────────────┐              ┌────────────────┐              ┌────────────┐
│   Order    │──publish────▶│  Message Broker │──subscribe──▶│Notification│
│  Service   │              │  (Kafka/SQS)   │              │  Service   │
└────────────┘              └────────────────┘              └────────────┘
                                    │
                                    │──subscribe──▶ ┌────────────┐
                                                    │ Analytics  │
                                                    │  Service   │
                                                    └────────────┘
```

### How an Order Flows Through Microservices

```
Customer places order
        │
        ▼
┌──────────────┐
│ API Gateway  │  (Route, Auth, Rate Limit)
└──────┬───────┘
       │
       ▼
┌──────────────┐    GET /users/123         ┌──────────────┐
│    Order     │──────────────────────────▶│    User      │
│   Service    │◀──────────────────────────│   Service    │
│              │    { name, address }       └──────────────┘
│              │
│              │    GET /products/456       ┌──────────────┐
│              │──────────────────────────▶│   Product    │
│              │◀──────────────────────────│   Service    │
│              │    { price, stock }        └──────────────┘
│              │
│              │    POST /payments          ┌──────────────┐
│              │──────────────────────────▶│   Payment    │
│              │◀──────────────────────────│   Service    │
│              │    { txn_id, status }      └──────────────┘
│              │
│              │    Event: "OrderCreated"   ┌──────────────┐
│              │──────────────────────────▶│    Kafka     │
└──────────────┘                           └──────┬───────┘
                                                  │
                              ┌────────────────────┼──────────────┐
                              ▼                    ▼              ▼
                     ┌──────────────┐    ┌──────────────┐  ┌───────────┐
                     │ Notification │    │  Inventory   │  │ Analytics │
                     │   Service    │    │   Service    │  │  Service  │
                     │ (send email) │    │(reduce stock)│  │(update DB)│
                     └──────────────┘    └──────────────┘  └───────────┘
```

### Service Discovery

Services need to **find each other** since their addresses can change (auto-scaling, failures, deployments):

```
┌────────────┐   1. Register        ┌─────────────────────┐
│   Order    │──────────────────────▶│  Service Registry   │
│  Service   │                       │  (Consul / Eureka)  │
│ 10.0.1.5   │                       │                     │
└────────────┘                       │  order-svc: 10.0.1.5│
                                     │  user-svc: 10.0.2.3 │
┌────────────┐   2. Where is        │  product-svc: 10.0.3│
│  Payment   │   "user-svc"?        │                     │
│  Service   │──────────────────────▶│                     │
│            │◀──────────────────────│  "10.0.2.3:8080"    │
│            │   3. Response         └─────────────────────┘
│            │
│            │   4. Call user-svc at 10.0.2.3:8080
│            │───────────────────────────────────▶ User Service
└────────────┘
```

### Data Ownership Pattern

```
WRONG (Shared Database):              RIGHT (Database per Service):

┌────────┐ ┌────────┐ ┌────────┐     ┌────────┐ ┌────────┐ ┌────────┐
│Order   │ │User    │ │Product │     │Order   │ │User    │ │Product │
│Service │ │Service │ │Service │     │Service │ │Service │ │Service │
└───┬────┘ └───┬────┘ └───┬────┘     └───┬────┘ └───┬────┘ └───┬────┘
    │          │          │               │          │          │
    ▼          ▼          ▼               ▼          ▼          ▼
┌─────────────────────────────┐     ┌────────┐ ┌────────┐ ┌────────┐
│     SHARED DATABASE         │     │Order DB│ │User DB │ │Prod DB │
│   (coupling! dangerous!)    │     │(Postgres)│ │(Mongo) │ │(Postgres)│
└─────────────────────────────┘     └────────┘ └────────┘ └────────┘
```

---

## Code Examples

### Python (FastAPI Microservice)

```python
# order_service/main.py — A small, focused microservice
from fastapi import FastAPI, HTTPException
import httpx  # Async HTTP client for calling other services
from pydantic import BaseModel

app = FastAPI(title="Order Service")

# Service URLs (in production, use service discovery)
USER_SERVICE = "http://user-service:8080"
PRODUCT_SERVICE = "http://product-service:8080"
PAYMENT_SERVICE = "http://payment-service:8080"

class OrderRequest(BaseModel):
    user_id: str
    product_id: str
    quantity: int

@app.post("/orders", status_code=201)
async def create_order(req: OrderRequest):
    async with httpx.AsyncClient(timeout=5.0) as client:
        # Call User Service to validate user exists
        user_resp = await client.get(f"{USER_SERVICE}/users/{req.user_id}")
        if user_resp.status_code != 200:
            raise HTTPException(404, "User not found")
        user = user_resp.json()
        
        # Call Product Service to check stock & get price
        prod_resp = await client.get(f"{PRODUCT_SERVICE}/products/{req.product_id}")
        if prod_resp.status_code != 200:
            raise HTTPException(404, "Product not found")
        product = prod_resp.json()
        
        if product["stock"] < req.quantity:
            raise HTTPException(400, "Insufficient stock")
        
        # Call Payment Service to charge
        total = product["price"] * req.quantity
        pay_resp = await client.post(f"{PAYMENT_SERVICE}/payments", json={
            "user_id": req.user_id, "amount": total
        })
        if pay_resp.status_code != 201:
            raise HTTPException(500, "Payment failed")
    
    # Save order to THIS service's own database
    order = await save_order(req.user_id, req.product_id, req.quantity, total)
    
    # Publish event (async — doesn't block response)
    await publish_event("order.created", {
        "order_id": order["id"], "user_email": user["email"]
    })
    
    return {"order_id": order["id"], "total": total, "status": "confirmed"}
```

### Java (Spring Boot Microservice with Resilience)

```java
// OrderService.java — Calls other microservices with resilience patterns
@Service
public class OrderService {
    
    private final WebClient.Builder webClientBuilder;
    private final OrderRepository orderRepo;
    private final KafkaTemplate<String, OrderEvent> kafka;
    
    @CircuitBreaker(name = "productService", fallbackMethod = "productFallback")
    @Retry(name = "productService")
    public Mono<OrderResponse> createOrder(OrderRequest request) {
        
        // Call Product Service (with circuit breaker)
        return webClientBuilder.build()
            .get()
            .uri("http://product-service/products/{id}", request.getProductId())
            .retrieve()
            .bodyToMono(ProductDto.class)
            .flatMap(product -> {
                if (product.getStock() < request.getQuantity()) {
                    return Mono.error(new InsufficientStockException());
                }
                
                BigDecimal total = product.getPrice()
                    .multiply(BigDecimal.valueOf(request.getQuantity()));
                
                // Call Payment Service
                return webClientBuilder.build()
                    .post()
                    .uri("http://payment-service/payments")
                    .bodyValue(new PaymentRequest(request.getUserId(), total))
                    .retrieve()
                    .bodyToMono(PaymentResponse.class)
                    .map(payment -> {
                        // Save to THIS service's DB
                        Order order = orderRepo.save(new Order(
                            request.getUserId(), request.getProductId(),
                            request.getQuantity(), total, payment.getTxnId()
                        ));
                        
                        // Publish event to Kafka
                        kafka.send("orders", new OrderEvent(
                            order.getId(), "CREATED", request.getUserId()
                        ));
                        
                        return new OrderResponse(order.getId(), total, "CONFIRMED");
                    });
            });
    }
    
    // Fallback when Product Service is down
    private Mono<OrderResponse> productFallback(OrderRequest req, Exception e) {
        return Mono.just(new OrderResponse(null, null, "SERVICE_UNAVAILABLE"));
    }
}
```

---

## Infrastructure Example

### Kubernetes Deployment

```yaml
# order-service/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3  # Scale independently from other services
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
        image: myregistry/order-service:v2.1.0
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: url
        - name: KAFKA_BROKERS
          value: "kafka-0:9092,kafka-1:9092,kafka-2:9092"
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
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

### Docker Compose for Local Development

```yaml
# docker-compose.yml — Run entire microservice ecosystem locally
version: '3.8'
services:
  # --- Services ---
  user-service:
    build: ./user-service
    ports: ["8081:8080"]
    environment:
      DATABASE_URL: mongodb://mongo:27017/users
  
  product-service:
    build: ./product-service
    ports: ["8082:8080"]
    environment:
      DATABASE_URL: postgresql://postgres:5432/products
  
  order-service:
    build: ./order-service
    ports: ["8083:8080"]
    environment:
      DATABASE_URL: postgresql://postgres:5432/orders
      USER_SERVICE_URL: http://user-service:8080
      PRODUCT_SERVICE_URL: http://product-service:8080
      KAFKA_BROKERS: kafka:9092
  
  # --- Infrastructure ---
  api-gateway:
    image: kong:latest
    ports: ["80:8000", "8443:8443"]
  
  mongo:
    image: mongo:6
    volumes: [mongo-data:/data/db]
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
  
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
  
  zookeeper:
    image: confluentinc/cp-zookeeper:latest

volumes:
  mongo-data:
```

---

## Real-World Example

### Netflix

Netflix is the **poster child** for microservices:

```
NETFLIX ARCHITECTURE (simplified):

┌──────────────────────────────────────────────────────┐
│                   ZUUL API GATEWAY                    │
└───────┬──────────┬──────────┬──────────┬─────────────┘
        │          │          │          │
        ▼          ▼          ▼          ▼
  ┌──────────┐ ┌────────┐ ┌─────────┐ ┌─────────────┐
  │ User     │ │Catalog │ │ Search  │ │Recommendation│
  │ Profile  │ │Service │ │ Service │ │   Service    │
  │ Service  │ │        │ │(Elastic │ │ (ML models)  │
  └──────────┘ └────────┘ │ search) │ └─────────────┘
                           └─────────┘
  ┌──────────┐ ┌────────┐ ┌─────────┐ ┌─────────────┐
  │Streaming │ │Billing │ │ A/B Test│ │   Playback  │
  │  Service │ │Service │ │ Service │ │   Service   │
  └──────────┘ └────────┘ └─────────┘ └─────────────┘
```

- **700+ microservices** in production
- Each team owns 1-5 services end-to-end
- Netflix invented many microservice tools: Eureka, Zuul, Hystrix, Ribbon
- They handle **250+ million subscribers** across 190 countries

### Uber

```
UBER's core services:

┌─────────────────────────────────────────┐
│            API GATEWAY                  │
└────────┬──────────┬──────────┬──────────┘
         ▼          ▼          ▼
   ┌──────────┐ ┌────────┐ ┌──────────┐
   │  Trip    │ │Matching│ │  Pricing │
   │ Service  │ │Service │ │  Service │
   │          │ │(assign │ │(surge,   │
   │          │ │driver) │ │ estimate)│
   └──────────┘ └────────┘ └──────────┘
   ┌──────────┐ ┌────────┐ ┌──────────┐
   │ Payment  │ │  Map   │ │Notification│
   │ Service  │ │Service │ │  Service  │
   └──────────┘ └────────┘ └──────────┘
```

- **4000+ microservices**
- Started as a monolith, migrated around 2014-2016
- Each service team has full autonomy (technology choice, deployment schedule)

---

## Common Mistakes / Pitfalls

### 1. Distributed Monolith
❌ **Mistake**: Services that are tightly coupled — you can't deploy one without deploying others.
✅ **Fix**: Services must be independently deployable. If Service A's deploy requires Service B to deploy simultaneously, they're not truly independent.

```
DISTRIBUTED MONOLITH (BAD):           TRUE MICROSERVICES (GOOD):
┌──────┐ ┌──────┐ ┌──────┐           ┌──────┐ ┌──────┐ ┌──────┐
│  A   │─│  B   │─│  C   │           │  A   │ │  B   │ │  C   │
│      │ │      │ │      │           │      │ │      │ │      │
│deploy│ │deploy│ │deploy│           │v1.2  │ │v3.0  │ │v1.0  │
│together│ │together│ │together│           │      │ │      │ │      │
└──────┘ └──────┘ └──────┘           └──────┘ └──────┘ └──────┘
                                      (each on own schedule)
```

### 2. Too Many Services Too Early
❌ **Mistake**: Splitting into 50 microservices on day one with 5 developers.
✅ **Fix**: Start with a monolith or 3-5 services. Split when you have clear domain boundaries AND enough team members.

### 3. No Observability
❌ **Mistake**: Can't trace a request across 10 services to find where it failed.
✅ **Fix**: Implement distributed tracing (Jaeger/Zipkin), centralized logging (ELK), and metrics (Prometheus/Grafana) from day one.

### 4. Shared Database
❌ **Mistake**: Two services reading/writing the same database table.
✅ **Fix**: Each service owns its data. Communicate through APIs or events.

### 5. Ignoring Network Failures
❌ **Mistake**: Calling other services without timeouts, retries, or circuit breakers.
✅ **Fix**: Implement resilience patterns (Chapter 12) — timeouts, retries with backoff, circuit breakers, fallbacks.

---

## When to Use / When NOT to Use

### ✅ Use Microservices When:

| Criteria | Why |
|---|---|
| **Large team (50+ developers)** | Independent teams, independent services |
| **Need independent scaling** | Scale hot services without scaling cold ones |
| **Need independent deployments** | Ship features without coordinating with other teams |
| **Complex domain with clear boundaries** | Each bounded context = one service |
| **Need technology diversity** | ML in Python, API in Java, real-time in Go |
| **High availability required** | One service failing shouldn't crash everything |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Small team (< 10 developers)** | Overhead is massive — you'll spend more time on infra than features |
| **Simple domain** | CRUD apps don't need 20 services |
| **No DevOps maturity** | Need CI/CD, monitoring, containers, service mesh |
| **Don't understand your domain** | You'll draw wrong boundaries and create a distributed monolith |
| **Strong consistency needed everywhere** | Distributed transactions are hard (Chapter 13.6) |

---

## Key Takeaways

- 🧩 **Microservices = small, independent services** that each own one business capability and their own data.
- 🔌 **Communication is over the network** (HTTP/gRPC/events) — this adds latency and failure modes that don't exist in monoliths.
- 📦 **Each service deploys independently** — different teams, different schedules, different technologies.
- 🗄️ **Each service owns its database** — no shared tables. Data is exchanged through APIs or events.
- ⚡ **The biggest benefit is organizational** — teams can move fast without stepping on each other.
- ⚠️ **The biggest cost is complexity** — distributed tracing, eventual consistency, network failures, deployment orchestration.
- 🎯 **You earn microservices** — start with a monolith, understand your domain, then extract services along natural boundaries.

---

## What's Next?

Microservices often need to react to things happening across the system. In **Chapter 4.6: Event-Driven Architecture (EDA)**, we'll explore how services can communicate by publishing and subscribing to events — enabling loose coupling and real-time reactions.
