# How Amazon/Flipkart E-Commerce Architecture Works

> **What you'll learn**: How the world's largest online stores handle 300+ million products, personalized recommendations for every user, massive flash sales (Prime Day/Big Billion Days), and process millions of orders per day — without crashing.

---

## Real-Life Analogy

Imagine a **shopping mall the size of a country**. Now imagine:

- **300 million different products** on shelves (more than any physical mall could ever hold)
- **500 million shoppers** browsing simultaneously during a sale
- Every shopper sees a **different arrangement of products** personalized just for them
- The checkout counter processes **thousands of transactions every second**
- Items are stored in **hundreds of warehouses** across the world, and the system picks the closest one
- If one section of the mall catches fire, the rest keeps working perfectly

No physical mall could do this. But Amazon and Flipkart do it digitally, every single day. Let's understand how.

---

## Core Concept Explained Step-by-Step

### Step 1: The Microservices Foundation

Amazon was one of the FIRST companies to adopt microservices (around 2002). Their entire platform is built as hundreds of independent services:

```
┌─────────────────────────────────────────────────────────────────────┐
│                 AMAZON'S MICROSERVICES LANDSCAPE                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ Product  │ │  Search  │ │  Cart    │ │  Order   │ │ Payment  │ │
│  │ Catalog  │ │  Service │ │  Service │ │  Service │ │ Service  │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│                                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ Inventory│ │  Pricing │ │ Recommend│ │ Shipping │ │  Review  │ │
│  │ Service  │ │  Service │ │ Service  │ │ Service  │ │ Service  │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│                                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  Auth    │ │ Warehouse│ │  Seller  │ │   Ad     │ │ Notifi-  │ │
│  │ Service  │ │  Service │ │  Service │ │  Service │ │ cation   │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│                                                                      │
│  ... 500+ more services ...                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 2: What Happens When You Visit Amazon.com

```
User opens amazon.com/product/12345
         │
         ▼
┌──────────────────┐
│  CloudFront CDN  │ ← Static assets (images, CSS, JS) served from edge
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  API Gateway /   │ ← Authentication, rate limiting, routing
│  Load Balancer   │
└────────┬─────────┘
         │
         ├──────────────────────────────────────────────┐
         │                                              │
         ▼                                              ▼
┌──────────────────┐                        ┌──────────────────┐
│ Product Service  │                        │ Recommendation   │
│                  │                        │ Service          │
│ • Title, desc   │                        │                  │
│ • Images        │                        │ • "You may also  │
│ • Specifications│                        │    like..."      │
│ • Variants      │                        │ • Based on your  │
└────────┬─────────┘                        │   history        │
         │                                  └──────────────────┘
         ├─────────────────────┐
         ▼                     ▼
┌──────────────────┐  ┌──────────────────┐
│ Pricing Service  │  │ Inventory Service│
│                  │  │                  │
│ • Dynamic price  │  │ • In stock?      │
│ • Seller offers  │  │ • Which warehouse│
│ • Coupons        │  │ • Delivery date  │
└──────────────────┘  └──────────────────┘
         │                     │
         ▼                     ▼
┌──────────────────────────────────────────┐
│           Page Assembly Service           │
│   Combines all data → renders HTML/JSON  │
└──────────────────────────────────────────┘
         │
         ▼
    User sees product page (< 200ms)
```

### Step 3: The Shopping Cart — Distributed State

The cart must work even if some services are down:

```
┌─────────────────────────────────────────────────────────┐
│              SHOPPING CART ARCHITECTURE                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  User adds item to cart                                  │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                       │
│  │ Cart Service  │ ── Writes to ──▶ DynamoDB            │
│  │ (Stateless)   │                  (Always Available)   │
│  └──────────────┘                                       │
│         │                                                │
│         │  Also publishes event                          │
│         ▼                                                │
│  ┌──────────────┐                                       │
│  │  Event Bus   │ ──▶ Recommendation Service (learns)   │
│  │  (Kinesis)   │ ──▶ Inventory Service (soft reserve)  │
│  └──────────────┘ ──▶ Analytics Service (tracking)      │
│                                                          │
│  KEY DESIGN: Cart uses "last-write-wins" conflict       │
│  resolution. Availability > Consistency here.            │
│  (See Chapter 13: CAP Theorem)                          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Step 4: Order Processing — The Critical Path

```
┌─────────────────────────────────────────────────────────────────┐
│                  ORDER PROCESSING PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  User clicks "Place Order"                                        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  Validate    │────▶│   Reserve    │────▶│   Process    │    │
│  │  Order       │     │   Inventory  │     │   Payment    │    │
│  └──────────────┘     └──────────────┘     └──────┬───────┘    │
│                                                     │            │
│                        ┌────────────────────────────┘            │
│                        │                                         │
│              Payment Success?                                     │
│                  │          │                                     │
│                 YES         NO                                    │
│                  │          │                                     │
│                  ▼          ▼                                     │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │  Confirm Order   │  │  Release         │                    │
│  │  (Write to DB)   │  │  Inventory       │                    │
│  └────────┬─────────┘  │  Notify User     │                    │
│           │             └──────────────────┘                    │
│           ▼                                                      │
│  ┌──────────────────────────────────────────┐                   │
│  │     ASYNC EVENTS (via SQS/Kinesis)       │                   │
│  │                                          │                   │
│  │  ──▶ Email Confirmation Service          │                   │
│  │  ──▶ Warehouse Picking Service           │                   │
│  │  ──▶ Seller Notification Service         │                   │
│  │  ──▶ Fraud Detection Service             │                   │
│  │  ──▶ Analytics Pipeline                  │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 5: Search and Discovery

```
┌─────────────────────────────────────────────────────────────────┐
│              PRODUCT SEARCH ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  User searches "wireless headphones under 2000"                  │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────┐                        │
│  │  Query Understanding                  │                        │
│  │  • Category: Electronics > Audio     │                        │
│  │  • Feature: wireless                 │                        │
│  │  • Price filter: < ₹2000            │                        │
│  └───────────────────┬──────────────────┘                        │
│                      │                                            │
│         ┌────────────┼────────────┐                              │
│         ▼            ▼            ▼                              │
│  ┌────────────┐ ┌──────────┐ ┌──────────────┐                   │
│  │ Search     │ │ Filter   │ │ Personalize  │                   │
│  │ Index      │ │ Service  │ │ & Rank       │                   │
│  │(Elastic-   │ │(Facets,  │ │(ML model for │                   │
│  │ search)    │ │ Price,   │ │ this user)   │                   │
│  │            │ │ Brand)   │ │              │                   │
│  └────────────┘ └──────────┘ └──────────────┘                   │
│         │            │            │                              │
│         └────────────┼────────────┘                              │
│                      ▼                                            │
│  ┌──────────────────────────────────────┐                        │
│  │  Merged & Ranked Results             │                        │
│  │  + Sponsored Products (Ads)          │                        │
│  │  + "Customers also bought..."        │                        │
│  └──────────────────────────────────────┘                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Amazon's DynamoDB — The Database Born from E-Commerce

Amazon created DynamoDB specifically because existing databases couldn't handle their shopping cart requirements. Key properties:

| Requirement | DynamoDB Solution |
|-------------|------------------|
| Always writable (even during partitions) | AP system, eventual consistency |
| Predictable latency | Single-digit millisecond at any scale |
| Auto-scaling | Adapts to traffic automatically |
| Global tables | Multi-region replication |

### Inventory Management at Scale

```
┌─────────────────────────────────────────────────────────────────┐
│           DISTRIBUTED INVENTORY SYSTEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Challenge: 300M products × multiple warehouses × real-time      │
│                                                                   │
│  ┌─────────────────────────────────────────────┐                 │
│  │        Inventory Record (per SKU)           │                 │
│  │                                             │                 │
│  │  SKU: HEADPHONE-WL-BK-001                  │                 │
│  │  ┌─────────────────────────────────────┐   │                 │
│  │  │ Warehouse Delhi:    150 units       │   │                 │
│  │  │ Warehouse Mumbai:    75 units       │   │                 │
│  │  │ Warehouse Bangalore: 200 units      │   │                 │
│  │  │ Warehouse Chennai:   50 units       │   │                 │
│  │  │ ─────────────────────────────       │   │                 │
│  │  │ Reserved (in carts):  30 units      │   │                 │
│  │  │ In transit:           20 units      │   │                 │
│  │  │ Available to promise: 425 units     │   │                 │
│  │  └─────────────────────────────────────┘   │                 │
│  └─────────────────────────────────────────────┘                 │
│                                                                   │
│  Consistency Strategy:                                            │
│  • Soft reserve when added to cart (optimistic)                  │
│  • Hard reserve when order placed (pessimistic lock)             │
│  • Release after 15 min if cart abandoned                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Flash Sale Architecture (Prime Day / Big Billion Days)

```
┌─────────────────────────────────────────────────────────────────┐
│              FLASH SALE ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Normal traffic: 10K requests/sec                                │
│  Flash sale:     1M+ requests/sec (100x spike)                   │
│                                                                   │
│  Strategy: Multiple defense layers                               │
│                                                                   │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Layer 1: CDN + Static Cache                           │      │
│  │ • Product images, descriptions pre-cached             │      │
│  │ • 80% of requests never reach origin                  │      │
│  └───────────────────────────────────────────────────────┘      │
│                       │                                          │
│                       ▼                                          │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Layer 2: Virtual Queue / Waiting Room                  │      │
│  │ • Users placed in queue ("You're #4523 in line")      │      │
│  │ • Rate limits requests to backend                      │      │
│  └───────────────────────────────────────────────────────┘      │
│                       │                                          │
│                       ▼                                          │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Layer 3: Pre-computed Inventory Tokens                 │      │
│  │ • Fixed number of "buy tokens" generated              │      │
│  │ • Once tokens exhausted → "Sold Out" (no DB hit)      │      │
│  └───────────────────────────────────────────────────────┘      │
│                       │                                          │
│                       ▼                                          │
│  ┌───────────────────────────────────────────────────────┐      │
│  │ Layer 4: Async Order Processing                        │      │
│  │ • Orders queued, processed asynchronously             │      │
│  │ • User gets "Order Received" immediately              │      │
│  │ • Confirmation email sent after processing            │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Recommendation Engine

```
┌─────────────────────────────────────────────────────────────────┐
│           RECOMMENDATION SYSTEM ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  35% of Amazon's revenue comes from recommendations!            │
│                                                                   │
│  ┌─────────────────────────────────────────────┐                 │
│  │           DATA COLLECTION                    │                 │
│  │  • What you viewed                          │                 │
│  │  • What you bought                          │                 │
│  │  • What you searched                        │                 │
│  │  • How long you spent on each page          │                 │
│  │  • What you added to cart but didn't buy    │                 │
│  └──────────────────────┬──────────────────────┘                 │
│                         │                                         │
│                         ▼                                         │
│  ┌─────────────────────────────────────────────┐                 │
│  │        ML MODELS (Multiple approaches)       │                 │
│  │                                             │                 │
│  │  1. Collaborative Filtering                 │                 │
│  │     "Users like you also bought..."         │                 │
│  │                                             │                 │
│  │  2. Content-Based Filtering                 │                 │
│  │     "Similar to items you've viewed"        │                 │
│  │                                             │                 │
│  │  3. Item-to-Item Similarity                 │                 │
│  │     "Frequently bought together"            │                 │
│  │                                             │                 │
│  │  4. Deep Learning (Neural Collaborative)    │                 │
│  │     Combines all signals for each user      │                 │
│  └──────────────────────┬──────────────────────┘                 │
│                         │                                         │
│                         ▼                                         │
│  ┌─────────────────────────────────────────────┐                 │
│  │     PRE-COMPUTED RESULTS (Offline)          │                 │
│  │     Stored in Redis/DynamoDB                │                 │
│  │     For each user: top 1000 recommendations │                 │
│  │     Updated every few hours                 │                 │
│  └─────────────────────────────────────────────┘                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Event-Driven Order Processing

```python
# Simplified e-commerce order processing using events
import json
import uuid
from datetime import datetime
from dataclasses import dataclass
from typing import List

@dataclass
class OrderItem:
    product_id: str
    quantity: int
    price: float

@dataclass
class Order:
    order_id: str
    user_id: str
    items: List[OrderItem]
    total: float
    status: str
    created_at: str

class OrderService:
    """Handles order placement with event-driven architecture."""
    
    def __init__(self, inventory_service, payment_service, event_bus):
        self.inventory = inventory_service
        self.payment = payment_service
        self.events = event_bus
    
    def place_order(self, user_id: str, items: List[OrderItem]) -> Order:
        order_id = str(uuid.uuid4())
        
        # Step 1: Reserve inventory (synchronous - critical path)
        for item in items:
            if not self.inventory.reserve(item.product_id, item.quantity):
                self.inventory.release_all(order_id)
                raise Exception(f"Item {item.product_id} out of stock")
        
        # Step 2: Process payment (synchronous - critical path)
        total = sum(item.price * item.quantity for item in items)
        payment_result = self.payment.charge(user_id, total, order_id)
        
        if not payment_result.success:
            self.inventory.release_all(order_id)
            raise Exception("Payment failed")
        
        # Step 3: Create order record
        order = Order(
            order_id=order_id, user_id=user_id, items=items,
            total=total, status="CONFIRMED",
            created_at=datetime.utcnow().isoformat()
        )
        
        # Step 4: Publish events (async - non-blocking)
        self.events.publish("order.confirmed", {
            "order_id": order_id,
            "user_id": user_id,
            "items": [vars(i) for i in items],
            "total": total
        })
        # Downstream services react: warehouse, email, analytics, fraud
        
        return order
```

### Java — Inventory Reservation with Optimistic Locking

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Simplified inventory service demonstrating how Amazon handles
 * concurrent purchases during flash sales.
 * Uses optimistic locking (CAS operations) to avoid bottlenecks.
 */
public class InventoryService {
    // SKU → available count (in production: distributed across DynamoDB)
    private final ConcurrentHashMap<String, AtomicInteger> stock = 
        new ConcurrentHashMap<>();

    public boolean reserveItem(String sku, int quantity) {
        AtomicInteger available = stock.get(sku);
        if (available == null) return false;
        
        // CAS (Compare-And-Swap) loop — lock-free concurrency
        while (true) {
            int current = available.get();
            if (current < quantity) {
                return false;  // Not enough stock
            }
            // Atomically decrement if value hasn't changed
            if (available.compareAndSet(current, current - quantity)) {
                return true;  // Successfully reserved!
            }
            // If CAS failed, another thread grabbed stock — retry
        }
    }

    public void releaseItem(String sku, int quantity) {
        stock.get(sku).addAndGet(quantity);
    }

    public void setStock(String sku, int quantity) {
        stock.put(sku, new AtomicInteger(quantity));
    }

    public static void main(String[] args) {
        InventoryService inventory = new InventoryService();
        inventory.setStock("HEADPHONE-001", 100);
        
        // Simulates flash sale: many threads trying to buy same item
        // CAS ensures no overselling without locks
        boolean reserved = inventory.reserveItem("HEADPHONE-001", 1);
        System.out.println("Reserved: " + reserved);  // true
    }
}
```

---

## Infrastructure Examples

### Amazon's Tech Stack

| Component | Technology |
|-----------|-----------|
| **CDN** | CloudFront (global edge network) |
| **API Gateway** | Custom internal gateway |
| **Compute** | EC2 + Lambda + ECS/EKS |
| **Primary DB** | DynamoDB (shopping cart, sessions) |
| **Search** | Custom (evolved from A9, uses Elasticsearch concepts) |
| **Cache** | ElastiCache (Redis) for sessions, recommendations |
| **Messaging** | SQS, SNS, Kinesis, EventBridge |
| **Storage** | S3 (product images, static assets) |
| **ML** | SageMaker (recommendations, fraud) |
| **Monitoring** | CloudWatch + internal tools |

### Flipkart's Tech Stack

| Component | Technology |
|-----------|-----------|
| **CDN** | Akamai + Custom |
| **Load Balancer** | Nginx + Custom L7 |
| **Compute** | Kubernetes on bare metal + cloud |
| **Primary DB** | MySQL (sharded) + Redis |
| **Search** | Elasticsearch (product search) |
| **Cache** | Redis + in-process (Caffeine) |
| **Messaging** | Kafka + RabbitMQ |
| **Storage** | Object storage for images |
| **ML** | Custom + Spark for recommendations |

---

## Real-World Example

### Amazon Prime Day 2023 — The Numbers

- **375 million items sold** in 48 hours
- **100,000+ transactions per second** at peak
- **Hundreds of millions of unique visitors**
- **Zero downtime** (despite 100x normal traffic)
- Pre-scaled to **3x expected peak** (just in case)

### Flipkart Big Billion Days

- **10x normal traffic** within first minutes
- **1 million concurrent users** on the app
- Uses a **"Waiting Room"** pattern to queue excess traffic
- **Separate clusters** for sale items vs regular shopping
- Pre-warms CDN, database connections, and application servers

### The Two-Pizza Team Rule

Amazon famously organizes engineering around the "Two-Pizza Team" rule:
- Each microservice is owned by a team small enough to feed with two pizzas (6-8 people)
- Each team owns their service end-to-end (build, deploy, operate)
- Teams communicate through APIs, not meetings
- This enables 500+ services to evolve independently

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Synchronous calls between all services | One slow service cascades failures to everything | Use async events for non-critical paths |
| Single database for everything | Becomes bottleneck at scale, can't specialize | Polyglot persistence: right DB for each service |
| No inventory reservation timeout | Items get locked forever in abandoned carts | Auto-release after 15-30 minutes |
| Showing real-time stock count to users | Causes thundering herd when stock is low | Show "In Stock" / "Only a few left" (fuzzy) |
| Monolithic search | Can't scale independently, slow to update | Dedicated search cluster (Elasticsearch) with async indexing |
| Not pre-scaling for sales events | Auto-scaling too slow for sudden 100x spikes | Pre-scale hours before event, verify capacity |

---

## When to Use / When NOT to Use

### When to Adopt This Architecture
- Millions of daily active users
- Hundreds of distinct business domains (catalog, payments, shipping)
- Multiple teams working in parallel (50+ engineers)
- Unpredictable traffic spikes (sales, viral products)
- Need to deploy multiple times per day

### When This is Overkill
- Early-stage startup (< 10K users) → Use a monolith
- Single product/simple catalog → Use Shopify or similar SaaS
- Small team (< 10 engineers) → Microservices add too much operational overhead
- Predictable, steady traffic → Don't need complex auto-scaling

---

## Key Takeaways

1. **Amazon pioneered microservices** — their architecture proves that hundreds of independent services can work together at massive scale
2. **DynamoDB was born from Amazon's needs** — the shopping cart required "always writable" availability, leading to an AP database design
3. **35% of revenue from recommendations** — ML-powered personalization is not optional at scale, it's the core business advantage
4. **Flash sales need multi-layer defense**: CDN cache → virtual queue → inventory tokens → async processing
5. **Event-driven architecture** decouples the critical path (order placement) from downstream concerns (email, analytics, warehouse)
6. **Inventory management** uses optimistic concurrency (CAS) for speed, with soft/hard reservation patterns to prevent overselling
7. **Two-Pizza Teams** enable organizational scaling — small autonomous teams owning independent services is key to moving fast at scale

---

## What's Next?

Next, we'll explore [How Netflix Streams to 200 Million Users](./03-netflix-streaming.md) — discovering how they deliver 15% of the world's internet bandwidth using a custom CDN, adaptive bitrate streaming, and chaos engineering.
