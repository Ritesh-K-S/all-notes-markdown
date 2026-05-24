# CQRS — Command Query Responsibility Segregation

> **What you'll learn**: How to separate read operations from write operations into different models, enabling independent optimization of each side for performance and scalability.

---

## Real-Life Analogy

Think of a **library**:

- **Writing** (adding new books): When a new book arrives, a librarian carefully catalogs it — recording the title, author, ISBN, shelf location, condition, acquisition date. This is a detailed, careful process with many validations.

- **Reading** (finding books): When a visitor wants to find a book, they use the **search catalog** — a simplified, optimized index. The catalog doesn't show acquisition dates or purchase orders. It shows title, author, and location. That's it.

The **cataloging system** (write model) and the **search catalog** (read model) are DIFFERENT. They store different data, are optimized for different purposes, and can be updated at different speeds.

That's CQRS — separate the system that **writes data** from the system that **reads data**, because reads and writes have fundamentally different requirements.

---

## Core Concept Explained Step-by-Step

### The Problem CQRS Solves

In a traditional system, the same model handles both reads and writes:

```
TRADITIONAL (Same model for read & write):

┌──────────┐  INSERT order        ┌────────────────────┐     ┌──────────┐
│  Client  │─────────────────────▶│  Order Service     │────▶│ Database │
│          │                      │  (same model)      │     │ (same    │
│          │  SELECT orders       │                    │     │  table)  │
│          │─────────────────────▶│  Complex queries   │────▶│          │
│          │◀─────────────────────│  fight with        │◀────│          │
└──────────┘  Slow response       │  complex writes    │     └──────────┘
                                  └────────────────────┘

Problem: The ORDER table is optimized for writes (normalized)
         but reads need JOINs across 10 tables = SLOW!
```

### The CQRS Solution

```
CQRS (Separate read & write models):

                    WRITE SIDE (Commands)          READ SIDE (Queries)
                    
┌──────────┐  "PlaceOrder"  ┌──────────────┐     ┌──────────────┐  "GetOrders"  ┌──────────┐
│  Client  │───────────────▶│   Command    │     │    Query     │◀──────────────│  Client  │
│          │                │   Handler    │     │   Handler    │              │          │
└──────────┘                └──────┬───────┘     └──────┬───────┘              └──────────┘
                                   │                    │
                                   ▼                    ▼
                           ┌──────────────┐      ┌──────────────┐
                           │  Write DB    │      │   Read DB    │
                           │ (Normalized) │      │(Denormalized)│
                           │              │      │              │
                           │ orders       │      │ order_views  │
                           │ order_items  │─sync─▶│ (flat table  │
                           │ products     │      │  pre-joined) │
                           │ customers    │      │              │
                           └──────────────┘      └──────────────┘
                           
                           Optimized for          Optimized for
                           consistency &          fast reads &
                           business rules         flexible queries
```

### Commands vs Queries

| Aspect | Command (Write) | Query (Read) |
|---|---|---|
| **Intent** | Change state | Retrieve state |
| **Example** | `PlaceOrder`, `CancelOrder`, `UpdateProfile` | `GetOrderById`, `ListUserOrders`, `SearchProducts` |
| **Validation** | Heavy (business rules, consistency checks) | Light (just auth + param validation) |
| **Return value** | Success/failure (maybe new ID) | Data (DTOs, views) |
| **Frequency** | Less frequent | Much more frequent (10:1 to 1000:1 ratio) |
| **Scaling need** | Moderate | Extreme |

---

## How It Works Internally

### The Synchronization Problem

If read and write databases are separate, how do they stay in sync?

```
SYNCHRONIZATION STRATEGIES:

1. SYNCHRONOUS (Dual Write):
   Command Handler ──write──▶ Write DB
                   ──write──▶ Read DB    (simple but risky — what if one fails?)

2. ASYNC via Events (Recommended):
   Command Handler ──write──▶ Write DB
                   ──publish─▶ Event Bus ──consume──▶ Read DB Projector
   
3. CHANGE DATA CAPTURE (CDC):
   Command Handler ──write──▶ Write DB
   Debezium ──reads DB log──────────────────────▶ Read DB Projector
```

### Event-Based Sync (Most Common)

```
┌────────────┐    ┌──────────────┐    ┌─────────────┐    ┌────────────────┐
│  Command   │    │  Write DB    │    │    Kafka    │    │  Projection    │
│  Handler   │    │ (PostgreSQL) │    │   (Events)  │    │   Builder      │
└─────┬──────┘    └──────────────┘    └──────┬──────┘    └───────┬────────┘
      │                                      │                    │
      │ 1. Save order                        │                    │
      │───────────────▶│                     │                    │
      │                │                     │                    │
      │ 2. Publish "OrderCreated"            │                    │
      │──────────────────────────────────────▶│                   │
      │                                      │                    │
      │                                      │ 3. Consume event   │
      │                                      │───────────────────▶│
      │                                      │                    │
      │                                      │     4. Update read │
      │                                      │        model       │
      │                                      │         ┌──────────▼──────┐
      │                                      │         │  Read DB        │
      │                                      │         │  (Elasticsearch │
      │                                      │         │   or Denorm'd   │
      │                                      │         │   Postgres)     │
      │                                      │         └─────────────────┘
```

### Read Model Examples

```
WRITE MODEL (Normalized — 3rd Normal Form):

┌─────────────┐    ┌──────────────┐    ┌────────────────┐
│   orders    │    │ order_items  │    │   products     │
├─────────────┤    ├──────────────┤    ├────────────────┤
│ id          │    │ id           │    │ id             │
│ customer_id │◀───│ order_id     │    │ name           │
│ status      │    │ product_id   │───▶│ price          │
│ created_at  │    │ quantity     │    │ category       │
└─────────────┘    │ unit_price   │    └────────────────┘
                   └──────────────┘

READ MODEL (Denormalized — optimized for display):

┌──────────────────────────────────────────┐
│            order_summary_view            │
├──────────────────────────────────────────┤
│ order_id                                 │
│ customer_name      ← pre-joined!         │
│ customer_email     ← pre-joined!         │
│ order_status                             │
│ total_amount       ← pre-calculated!     │
│ item_count         ← pre-calculated!     │
│ items_json         ← embedded array!     │
│ created_at                               │
│ last_updated                             │
└──────────────────────────────────────────┘

Query: SELECT * FROM order_summary_view WHERE customer_id = ?
(No JOINs! No calculations! Instant response!)
```

---

## Code Examples

### Python (CQRS Implementation)

```python
# === COMMAND SIDE ===
# commands.py — Define what actions can change state
from dataclasses import dataclass

@dataclass
class PlaceOrderCommand:
    customer_id: str
    items: list  # [{"product_id": "x", "quantity": 2}]

@dataclass
class CancelOrderCommand:
    order_id: str
    reason: str

# command_handlers.py — Process commands (write side)
class OrderCommandHandler:
    def __init__(self, write_db, event_publisher):
        self.db = write_db
        self.events = event_publisher
    
    def handle_place_order(self, cmd: PlaceOrderCommand):
        # Validate business rules (write side has complex validation)
        for item in cmd.items:
            product = self.db.get_product(item["product_id"])
            if product.stock < item["quantity"]:
                raise InsufficientStockError(product.id)
        
        # Write to the write database
        order = Order(customer_id=cmd.customer_id, items=cmd.items)
        self.db.save_order(order)
        
        # Publish event to sync read model
        self.events.publish("OrderCreated", {
            "order_id": order.id,
            "customer_id": cmd.customer_id,
            "items": cmd.items,
            "total": order.total,
            "created_at": order.created_at.isoformat()
        })
        
        return order.id

# === QUERY SIDE ===
# query_handlers.py — Process queries (read side)
class OrderQueryHandler:
    def __init__(self, read_db):
        self.read_db = read_db  # Could be Elasticsearch, Redis, denormalized PG
    
    def get_orders_for_customer(self, customer_id: str, page: int = 1):
        # Simple, fast read from denormalized view
        return self.read_db.query(
            "SELECT * FROM order_summary_view WHERE customer_id = %s "
            "ORDER BY created_at DESC LIMIT 20 OFFSET %s",
            (customer_id, (page - 1) * 20)
        )
    
    def search_orders(self, query: str):
        # Full-text search on the read model (Elasticsearch)
        return self.read_db.search(index="orders", body={
            "query": {"multi_match": {"query": query, "fields": ["*"]}}
        })

# === PROJECTION BUILDER (syncs read model) ===
# projections.py — Consume events and update read database
class OrderProjection:
    def __init__(self, read_db):
        self.read_db = read_db
    
    def on_order_created(self, event):
        """Build/update the denormalized read model from events."""
        self.read_db.execute(
            "INSERT INTO order_summary_view "
            "(order_id, customer_id, total, item_count, status, created_at) "
            "VALUES (%s, %s, %s, %s, %s, %s)",
            (event["order_id"], event["customer_id"], event["total"],
             len(event["items"]), "PENDING", event["created_at"])
        )
```

### Java (Spring Boot CQRS)

```java
// === COMMAND SIDE ===
// PlaceOrderCommand.java
public record PlaceOrderCommand(
    String customerId,
    List<OrderItem> items
) {}

// OrderCommandHandler.java
@Service
public class OrderCommandHandler {
    
    @Autowired private OrderWriteRepository writeRepo;
    @Autowired private KafkaTemplate<String, DomainEvent> eventPublisher;
    
    @Transactional
    public String handlePlaceOrder(PlaceOrderCommand cmd) {
        // Complex validation on write side
        validateStock(cmd.items());
        validateCustomerCredit(cmd.customerId());
        
        // Persist to write database (normalized)
        Order order = new Order(cmd.customerId(), cmd.items());
        writeRepo.save(order);
        
        // Publish domain event (triggers read model update)
        eventPublisher.send("domain-events", new OrderCreatedEvent(
            order.getId(), cmd.customerId(), order.getTotal(), cmd.items()
        ));
        
        return order.getId();
    }
}

// === QUERY SIDE ===
// OrderQueryHandler.java
@Service
public class OrderQueryHandler {
    
    @Autowired private OrderReadRepository readRepo; // Elasticsearch or denorm'd table
    
    public List<OrderSummaryDto> getCustomerOrders(String customerId, int page) {
        // Fast read from denormalized view — no JOINs!
        return readRepo.findByCustomerId(customerId, PageRequest.of(page, 20));
    }
    
    public List<OrderSummaryDto> searchOrders(String query) {
        return readRepo.searchFullText(query);
    }
}

// === PROJECTION (Event Consumer) ===
@Component
public class OrderProjectionBuilder {
    
    @Autowired private OrderReadRepository readRepo;
    
    @KafkaListener(topics = "domain-events", groupId = "order-projection")
    public void onEvent(DomainEvent event) {
        if (event instanceof OrderCreatedEvent e) {
            readRepo.save(new OrderSummaryView(
                e.getOrderId(), e.getCustomerId(), e.getTotal(),
                e.getItems().size(), "PENDING", Instant.now()
            ));
        } else if (event instanceof OrderShippedEvent e) {
            readRepo.updateStatus(e.getOrderId(), "SHIPPED");
        }
    }
}
```

---

## Infrastructure Example

### CQRS with Different Databases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CQRS INFRASTRUCTURE                              │
│                                                                         │
│  WRITE PATH:                                                            │
│  ┌──────────┐    ┌───────────────┐    ┌──────────────────────┐          │
│  │  Client  │───▶│ Command API   │───▶│ PostgreSQL (Write DB)│          │
│  │  (POST,  │    │ (validates,   │    │ - Normalized tables  │          │
│  │   PUT,   │    │  saves,       │    │ - ACID transactions  │          │
│  │  DELETE)  │    │  publishes)   │    │ - Foreign keys       │          │
│  └──────────┘    └───────┬───────┘    └──────────────────────┘          │
│                          │                                              │
│                          │ publish event                                │
│                          ▼                                              │
│                   ┌──────────────┐                                      │
│                   │    Kafka     │                                      │
│                   │  (Events)    │                                      │
│                   └──────┬───────┘                                      │
│                          │                                              │
│                          │ consume                                      │
│                          ▼                                              │
│  READ PATH:      ┌──────────────┐    ┌──────────────────────┐          │
│  ┌──────────┐    │  Projection  │───▶│ Elasticsearch (Read) │          │
│  │  Client  │───▶│  Builder     │    │ - Denormalized docs  │          │
│  │  (GET,   │    └──────────────┘    │ - Full-text search   │          │
│  │  search) │                        │ - Fast aggregations  │          │
│  │          │───────────────────────▶│                      │          │
│  └──────────┘    Query API           └──────────────────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scaling Reads Independently

```
WRITE SIDE (1 instance is fine):     READ SIDE (scale as needed):

┌──────────────────┐                 ┌──────────────────┐
│ Command Handler  │                 │  Query Handler   │ x 10 replicas
│    (1 instance)  │                 │  (10 instances)  │
└────────┬─────────┘                 └────────┬─────────┘
         │                                    │
         ▼                                    ▼
┌──────────────────┐                 ┌──────────────────┐
│  PostgreSQL      │                 │  Elasticsearch   │
│  (1 primary +    │                 │  (3 nodes,       │
│   1 standby)     │                 │   6 shards,      │
└──────────────────┘                 │   2 replicas)    │
                                     └──────────────────┘

Writes: ~100/sec (OK)                Reads: ~100,000/sec (needed!)
```

---

## Real-World Example

### Twitter's Timeline

Twitter's timeline is a classic CQRS example:

```
WRITE SIDE:                              READ SIDE:
User posts tweet                         User opens timeline

┌──────────┐  POST /tweets  ┌────────┐   ┌──────────┐  GET /timeline  ┌───────────┐
│  User    │───────────────▶│ Tweet  │   │  User    │────────────────▶│ Timeline  │
│          │                │Service │   │          │                  │ Service   │
└──────────┘                └───┬────┘   └──────────┘                  └─────┬─────┘
                                │                                            │
                                ▼                                            ▼
                         ┌────────────┐                              ┌────────────────┐
                         │ Tweet DB   │                              │  Timeline Cache │
                         │(write-opt) │                              │  (Redis)        │
                         └────────────┘                              │                 │
                                │                                    │  User 123:      │
                                │ Fan-out event                      │  [tweet_a,      │
                                ▼                                    │   tweet_b,      │
                         ┌────────────┐                              │   tweet_c]      │
                         │  Fan-out   │──────────────────────────────▶                 │
                         │  Service   │  Pre-compute & cache         └────────────────┘
                         └────────────┘  timelines for each follower
```

### Amazon Product Pages

```
WRITE (Seller updates product):     READ (Customer views product):

Detailed product data               Pre-rendered product view
+ Inventory management              + Review summaries
+ Pricing rules                     + "Frequently bought together"
+ Compliance checks                 + Price display
                                    + Delivery estimates
         │                                    │
         ▼                                    ▼
  PostgreSQL (normalized)           DynamoDB + Elasticsearch
  (strong consistency)              (eventual consistency, FAST reads)
```

---

## Common Mistakes / Pitfalls

### 1. Eventual Consistency Confusion
❌ **Mistake**: User creates an order, immediately refreshes, and doesn't see it (read model hasn't synced yet).
✅ **Fix**: After a write, redirect to a page that reads from the write DB, or use "read-your-own-writes" consistency.

```
FIX: Read-Your-Own-Writes

POST /orders → returns order_id: "ORD-123"
                         │
GET /orders/ORD-123 ─────┘ (immediately after)
    → Route to WRITE DB for this specific read
    → User sees their own order instantly
    → Other users see it from READ DB (eventually)
```

### 2. Over-Engineering Simple Systems
❌ **Mistake**: Using CQRS for a basic CRUD app with 100 users.
✅ **Fix**: Only use CQRS when read/write patterns are genuinely different and scaling them together is problematic.

### 3. Forgetting to Handle Projection Failures
❌ **Mistake**: Event processing fails, read model becomes permanently stale.
✅ **Fix**: Implement dead letter queues, monitoring on projection lag, and ability to rebuild read model from events.

### 4. Too Many Read Models
❌ **Mistake**: Creating a separate read model for every query variation.
✅ **Fix**: Design a few flexible read models that serve multiple query patterns.

### 5. Not Monitoring Sync Lag
❌ **Mistake**: Read model is 5 minutes behind write model, and nobody notices.
✅ **Fix**: Monitor projection lag. Alert when it exceeds acceptable thresholds (e.g., > 5 seconds).

---

## When to Use / When NOT to Use

### ✅ Use CQRS When:

| Criteria | Why |
|---|---|
| **Reads vastly outnumber writes** (100:1+) | Scale read side independently |
| **Read and write models are very different** | Different structures for different needs |
| **Complex queries slow down writes** | Separate them to avoid contention |
| **Multiple read representations needed** | Search index + dashboard + API response |
| **Event-driven system already in place** | CQRS is a natural fit with events |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Simple CRUD** | Massive over-engineering |
| **Need instant consistency** | Eventual consistency adds complexity |
| **Small data volume** | Single DB handles both reads and writes fine |
| **Team not experienced with async** | Debugging projection issues is hard |
| **Read and write patterns are similar** | No benefit from separation |

---

## Key Takeaways

- ✂️ **CQRS separates the write model from the read model** — each can use different databases, schemas, and optimization strategies.
- 📖 **Read model is denormalized** — pre-joined, pre-calculated data for instant query responses with no JOINs.
- ✍️ **Write model is normalized** — optimized for consistency, validation, and business rules.
- 🔄 **Synchronization happens via events** — write side publishes events, projection builders update the read side.
- ⏱️ **Eventual consistency is the trade-off** — the read model may lag behind the write model by milliseconds to seconds.
- 📈 **Scale reads and writes independently** — 1 write instance + 100 read replicas is perfectly valid.
- 🎯 **Don't use CQRS everywhere** — only where read/write asymmetry justifies the added complexity.

---

## What's Next?

CQRS pairs beautifully with a pattern that stores state as a sequence of events. In **Chapter 4.8: Event Sourcing**, we'll learn how to store every change as an immutable event — giving us a complete audit trail, time-travel debugging, and the ability to rebuild any state from scratch.
