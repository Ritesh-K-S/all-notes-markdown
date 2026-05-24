# Hexagonal Architecture (Ports & Adapters)

> **What you'll learn**: How to structure your application so that business logic is completely independent of frameworks, databases, and external services — making it testable, flexible, and resistant to technology changes.

---

## Real-Life Analogy

Think of a **power outlet (socket) and a plug adapter**:

Your laptop doesn't care if the electricity comes from a coal plant, solar panel, or wind turbine. It also doesn't care if you're in the US (110V) or Europe (220V) — as long as the right **adapter** (converter) is plugged in.

Your laptop defines what it NEEDS (a certain voltage and connector shape = the **port**), and different **adapters** convert whatever is available in the real world to match that need.

Similarly, in hexagonal architecture:
- Your **business logic** (the laptop) defines what it needs via **ports** (interfaces)
- **Adapters** connect the real world (databases, web frameworks, APIs) to those ports
- The business logic NEVER knows about the real world — it only talks through ports

---

## Core Concept Explained Step-by-Step

### The Core Idea

Your application is a **hexagon** (the shape is symbolic, representing multiple sides/ports). The inside contains pure business logic. The outside has adapters that connect to the real world.

```
                    ┌──────────────────────────────────────┐
                    │          OUTSIDE WORLD               │
                    │                                      │
        ┌───────────────────┐            ┌───────────────────────┐
        │   REST Controller │            │   PostgreSQL Adapter  │
        │   (Driving Adapter)│            │   (Driven Adapter)    │
        └────────┬──────────┘            └──────────┬────────────┘
                 │                                   │
                 │ uses Port                  implements Port
                 ▼                                   ▼
        ╔════════════════════════════════════════════════════╗
        ║         ┌──────────────────────┐                  ║
        ║         │      PORT            │                  ║
        ║         │ (Input Interface)    │                  ║
        ║         └──────────┬───────────┘                  ║
        ║                    │                              ║
        ║                    ▼                              ║
        ║    ┌───────────────────────────────────┐          ║
        ║    │                                   │          ║
        ║    │        DOMAIN / BUSINESS          │          ║
        ║    │            LOGIC                  │          ║
        ║    │                                   │          ║
        ║    │   (Pure, no framework imports,    │          ║
        ║    │    no DB queries, no HTTP)        │          ║
        ║    │                                   │          ║
        ║    └───────────────────────────────────┘          ║
        ║                    │                              ║
        ║         ┌─────────▼────────────┐                  ║
        ║         │      PORT            │                  ║
        ║         │ (Output Interface)   │                  ║
        ║         └──────────────────────┘                  ║
        ╚════════════════════════════════════════════════════╝
                 ▲                                   ▲
                 │                                   │
        ┌────────┴──────────┐            ┌──────────┴────────────┐
        │   CLI Adapter     │            │   MongoDB Adapter     │
        │  (Driving Adapter) │            │   (Driven Adapter)    │
        └───────────────────┘            └───────────────────────┘
```

### Key Terminology

| Term | Meaning | Example |
|---|---|---|
| **Port** | An interface defined by the business logic | `OrderRepository` interface, `PaymentGateway` interface |
| **Adapter** | Implementation that connects a port to the real world | `PostgresOrderRepository`, `StripePaymentGateway` |
| **Driving Adapter** | Triggers the application (input) | REST controller, CLI command, message consumer |
| **Driven Adapter** | Called by the application (output) | Database client, HTTP client, email sender |
| **Domain** | Pure business logic — no external dependencies | `Order`, `OrderService`, business rules |

### Driving vs Driven

```
DRIVING (Input/Primary):                DRIVEN (Output/Secondary):
"What triggers the application?"        "What does the application call out to?"

┌──────────────────┐                    ┌──────────────────┐
│ REST API         │───┐            ┌───│ PostgreSQL       │
│ GraphQL          │───┤            ├───│ MongoDB          │
│ CLI              │───┤  ┌──────┐  ├───│ Redis            │
│ Message Consumer │───┼─▶│Domain│──┼───│ Kafka            │
│ Scheduled Job    │───┤  └──────┘  ├───│ Stripe API       │
│ gRPC             │───┤            ├───│ SendGrid (email) │
│ WebSocket        │───┘            └───│ S3 (file storage)│
└──────────────────┘                    └──────────────────┘

Domain NEVER imports from the outside world.
It defines ports (interfaces) that adapters implement.
```

---

## How It Works Internally

### Dependency Rule

The most critical rule: **dependencies point INWARD**.

```
DEPENDENCY DIRECTION:

Adapters ──depend on──▶ Ports ──defined by──▶ Domain

NEVER:
Domain ──✗──▶ Adapters
Domain ──✗──▶ Frameworks
Domain ──✗──▶ Database libraries
```

### Layer Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     ADAPTERS (Outermost)                         │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐     │
│  │ REST API    │  │ Kafka Consumer│  │PostgresOrderRepo   │     │
│  │ (FastAPI /  │  │ (adapter)     │  │(implements         │     │
│  │  Spring)    │  │               │  │ OrderRepository)   │     │
│  └──────┬──────┘  └──────┬───────┘  └─────────┬──────────┘     │
│         │                │                     │                │
├─────────┼────────────────┼─────────────────────┼────────────────┤
│         │         PORTS (Interfaces)            │                │
│         │                │                     │                │
│  ┌──────▼──────┐  ┌─────▼───────┐  ┌──────────▼──────────┐     │
│  │CreateOrder  │  │HandleEvent  │  │  OrderRepository    │     │
│  │UseCase      │  │UseCase      │  │  (interface)        │     │
│  │(input port) │  │(input port) │  │  (output port)      │     │
│  └──────┬──────┘  └─────┬───────┘  └──────────▲──────────┘     │
│         │                │                     │                │
├─────────┼────────────────┼─────────────────────┼────────────────┤
│         │          DOMAIN (Innermost)           │                │
│         │                │                     │                │
│         ▼                ▼                     │                │
│  ┌──────────────────────────────────────────┐  │                │
│  │                                          │  │                │
│  │  Order (entity)                          │  │                │
│  │  OrderService (uses OrderRepository port)│──┘                │
│  │  PricingRules (pure logic)               │                   │
│  │  DiscountCalculator (pure logic)         │                   │
│  │                                          │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why "Hexagonal"?

The hexagon shape represents that there are **multiple ports** — your application can be driven by many different inputs and can talk to many different outputs. It's not just web → app → database (layered). It's a richer model.

---

## Code Examples

### Python (Hexagonal Architecture)

```python
# === DOMAIN LAYER (innermost — NO external imports!) ===

# domain/models.py — Pure domain entities
class Order:
    def __init__(self, customer_id: str, items: list):
        self.id = None  # Set by repository
        self.customer_id = customer_id
        self.items = items
        self.status = "PENDING"
        self.total = sum(item['price'] * item['quantity'] for item in items)
    
    def confirm(self):
        if self.status != "PENDING":
            raise DomainError(f"Cannot confirm order in status {self.status}")
        self.status = "CONFIRMED"
    
    def cancel(self, reason: str):
        if self.status == "SHIPPED":
            raise DomainError("Cannot cancel shipped order")
        self.status = "CANCELLED"
        self.cancel_reason = reason

# domain/ports.py — Interfaces (what the domain NEEDS from the outside)
from abc import ABC, abstractmethod

class OrderRepository(ABC):
    """Output port: domain needs to persist orders."""
    @abstractmethod
    def save(self, order: Order) -> Order: ...
    
    @abstractmethod
    def find_by_id(self, order_id: str) -> Order: ...

class PaymentGateway(ABC):
    """Output port: domain needs to charge payments."""
    @abstractmethod
    def charge(self, customer_id: str, amount: float) -> str: ...

class NotificationSender(ABC):
    """Output port: domain needs to send notifications."""
    @abstractmethod
    def send_order_confirmation(self, email: str, order_id: str): ...

# domain/services.py — Business logic (uses ports, not implementations)
class OrderService:
    """Application service — orchestrates domain logic."""
    
    def __init__(self, order_repo: OrderRepository, 
                 payment: PaymentGateway, 
                 notifier: NotificationSender):
        # Depends on INTERFACES, not concrete implementations
        self.order_repo = order_repo
        self.payment = payment
        self.notifier = notifier
    
    def place_order(self, customer_id: str, items: list, email: str) -> str:
        # Pure business logic — no Flask, no SQL, no Stripe
        order = Order(customer_id, items)
        
        # Charge payment (through port — doesn't know if it's Stripe or PayPal)
        txn_id = self.payment.charge(customer_id, order.total)
        order.confirm()
        
        # Persist (through port — doesn't know if it's Postgres or Mongo)
        saved_order = self.order_repo.save(order)
        
        # Notify (through port — doesn't know if it's email or SMS)
        self.notifier.send_order_confirmation(email, saved_order.id)
        
        return saved_order.id


# === ADAPTERS (outermost — connect real world to ports) ===

# adapters/persistence/postgres_order_repo.py — Driven adapter
import psycopg2

class PostgresOrderRepository(OrderRepository):
    """Adapter: implements OrderRepository port using PostgreSQL."""
    
    def __init__(self, connection_string: str):
        self.conn = psycopg2.connect(connection_string)
    
    def save(self, order: Order) -> Order:
        cursor = self.conn.cursor()
        cursor.execute(
            "INSERT INTO orders (customer_id, total, status) VALUES (%s, %s, %s) RETURNING id",
            (order.customer_id, order.total, order.status)
        )
        order.id = cursor.fetchone()[0]
        self.conn.commit()
        return order
    
    def find_by_id(self, order_id: str) -> Order:
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM orders WHERE id = %s", (order_id,))
        row = cursor.fetchone()
        return self._to_domain(row)

# adapters/payment/stripe_adapter.py — Driven adapter
import stripe

class StripePaymentGateway(PaymentGateway):
    """Adapter: implements PaymentGateway port using Stripe."""
    
    def __init__(self, api_key: str):
        stripe.api_key = api_key
    
    def charge(self, customer_id: str, amount: float) -> str:
        charge = stripe.Charge.create(
            amount=int(amount * 100),  # Stripe uses cents
            currency="usd",
            customer=customer_id
        )
        return charge.id

# adapters/web/flask_api.py — Driving adapter
from flask import Flask, request, jsonify

app = Flask(__name__)

# Wire everything together (dependency injection)
order_service = OrderService(
    order_repo=PostgresOrderRepository("postgresql://localhost/shop"),
    payment=StripePaymentGateway("sk_live_xxx"),
    notifier=SendGridNotificationSender("sg_api_key")
)

@app.route('/api/orders', methods=['POST'])
def create_order():
    """Driving adapter: translates HTTP into domain call."""
    data = request.json
    order_id = order_service.place_order(
        customer_id=data['customer_id'],
        items=data['items'],
        email=data['email']
    )
    return jsonify({"order_id": order_id}), 201
```

### Java (Spring Boot Hexagonal)

```java
// === DOMAIN (No Spring annotations! No JPA! Pure Java!) ===

// domain/model/Order.java
public class Order {
    private String id;
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal total;
    private OrderStatus status;
    
    public Order(String customerId, List<OrderItem> items) {
        this.customerId = customerId;
        this.items = items;
        this.total = items.stream()
            .map(i -> i.price().multiply(BigDecimal.valueOf(i.quantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        this.status = OrderStatus.PENDING;
    }
    
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new DomainException("Cannot confirm order in status: " + status);
        }
        this.status = OrderStatus.CONFIRMED;
    }
}

// domain/port/output/OrderRepository.java — Output port
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(String id);
}

// domain/port/output/PaymentGateway.java — Output port
public interface PaymentGateway {
    String charge(String customerId, BigDecimal amount);
}

// domain/port/input/CreateOrderUseCase.java — Input port
public interface CreateOrderUseCase {
    String execute(CreateOrderCommand command);
}

// domain/service/OrderService.java — Implements input port, uses output ports
public class OrderService implements CreateOrderUseCase {
    private final OrderRepository orderRepo;     // Output port
    private final PaymentGateway paymentGateway;  // Output port
    
    public OrderService(OrderRepository orderRepo, PaymentGateway paymentGateway) {
        this.orderRepo = orderRepo;
        this.paymentGateway = paymentGateway;
    }
    
    @Override
    public String execute(CreateOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        paymentGateway.charge(cmd.customerId(), order.getTotal());
        order.confirm();
        Order saved = orderRepo.save(order);
        return saved.getId();
    }
}

// === ADAPTERS ===

// adapter/output/persistence/JpaOrderRepository.java — Driven adapter
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Autowired private SpringDataOrderRepo springRepo;
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = OrderMapper.toEntity(order);
        OrderEntity saved = springRepo.save(entity);
        return OrderMapper.toDomain(saved);
    }
}

// adapter/input/web/OrderController.java — Driving adapter
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final CreateOrderUseCase createOrder; // Input port
    
    @PostMapping
    public ResponseEntity<?> create(@RequestBody CreateOrderRequest req) {
        String orderId = createOrder.execute(
            new CreateOrderCommand(req.getCustomerId(), req.getItems())
        );
        return ResponseEntity.status(201).body(Map.of("order_id", orderId));
    }
}
```

### Project Structure

```
src/
├── domain/                     ← INNERMOST (no external deps!)
│   ├── model/
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   └── OrderStatus.java
│   ├── port/
│   │   ├── input/              ← Input ports (use cases)
│   │   │   └── CreateOrderUseCase.java
│   │   └── output/             ← Output ports (driven)
│   │       ├── OrderRepository.java
│   │       └── PaymentGateway.java
│   ├── service/
│   │   └── OrderService.java   ← Implements input ports
│   └── exception/
│       └── DomainException.java
│
├── adapter/                    ← OUTERMOST (connects to real world)
│   ├── input/                  ← Driving adapters
│   │   ├── web/
│   │   │   └── OrderController.java
│   │   ├── cli/
│   │   │   └── OrderCLI.java
│   │   └── messaging/
│   │       └── KafkaOrderConsumer.java
│   └── output/                 ← Driven adapters
│       ├── persistence/
│       │   ├── JpaOrderRepository.java
│       │   └── MongoOrderRepository.java
│       ├── payment/
│       │   ├── StripePaymentGateway.java
│       │   └── PayPalPaymentGateway.java
│       └── notification/
│           └── SendGridEmailSender.java
│
└── config/                     ← Wiring (dependency injection)
    └── AppConfig.java
```

---

## Infrastructure Example

### Swapping Adapters Without Changing Business Logic

```
DEVELOPMENT:                    PRODUCTION:                   TESTING:

┌─────────────────┐            ┌─────────────────┐           ┌──────────────────┐
│ Domain Logic    │            │ Domain Logic    │           │ Domain Logic     │
│ (SAME code!)    │            │ (SAME code!)    │           │ (SAME code!)     │
└────────┬────────┘            └────────┬────────┘           └────────┬─────────┘
         │                              │                             │
         ▼                              ▼                             ▼
┌─────────────────┐            ┌─────────────────┐           ┌──────────────────┐
│ H2 In-Memory DB │            │ PostgreSQL RDS  │           │ InMemory HashMap │
│ Fake Payment    │            │ Stripe API      │           │ Fake Payment     │
│ Console Logger  │            │ SendGrid Email  │           │ Spy Notifier     │
└─────────────────┘            └─────────────────┘           └──────────────────┘

Same domain code, different adapters = different environments!
```

### Testing Benefits

```python
# Unit test — ZERO external dependencies (no DB, no HTTP, no Stripe!)
def test_place_order_charges_correct_amount():
    # Arrange: use fake adapters
    fake_repo = InMemoryOrderRepository()
    fake_payment = FakePaymentGateway()
    fake_notifier = SpyNotificationSender()
    
    service = OrderService(fake_repo, fake_payment, fake_notifier)
    
    # Act
    order_id = service.place_order(
        customer_id="cust-1",
        items=[{"product": "Widget", "price": 29.99, "quantity": 2}],
        email="alice@test.com"
    )
    
    # Assert
    assert fake_payment.last_charge_amount == 59.98
    assert fake_notifier.emails_sent == 1
    assert fake_repo.find_by_id(order_id).status == "CONFIRMED"

# No database setup, no API mocking, runs in milliseconds!
```

---

## Real-World Example

### Netflix — Clean Architecture for Microservices

Netflix applies hexagonal principles within each microservice:
- **Domain logic** is pure Java (no framework annotations)
- **Adapters** connect to Cassandra, Memcached, other services
- They can swap data stores per service without touching business logic

### Spotify — Backend Services

Spotify's backend services use hexagonal architecture:
- Music recommendation domain logic is pure (ML algorithms)
- Adapters connect to different data sources (listening history, social graph)
- Can swap from one ML framework to another without touching domain

### The "Ports & Adapters" Names Used in Industry

| Name | Same Concept |
|---|---|
| **Hexagonal Architecture** | Original name (Alistair Cockburn, 2005) |
| **Ports & Adapters** | Alternative name (same thing) |
| **Clean Architecture** | Robert C. Martin's version (similar principles) |
| **Onion Architecture** | Jeffrey Palermo's version (similar principles) |

---

## Common Mistakes / Pitfalls

### 1. Domain Imports Framework Code
❌ **Mistake**: `@Entity`, `@Table`, `@Autowired` in domain classes.
✅ **Fix**: Domain classes are POJOs (plain objects). No framework annotations. Use separate entity classes in the adapter layer.

```java
// BAD: Domain polluted with JPA
@Entity @Table(name = "orders")  // ← Framework leak!
public class Order {
    @Id @GeneratedValue  // ← Framework leak!
    private Long id;
}

// GOOD: Domain is pure
public class Order {
    private String id;  // Plain Java, no annotations
}

// Adapter has its own entity
@Entity @Table(name = "orders")
public class OrderEntity {  // Separate class for persistence
    @Id @GeneratedValue
    private Long id;
}
// + Mapper to convert between them
```

### 2. Ports Too Granular or Too Broad
❌ **Mistake**: One port per method, or one giant port for everything.
✅ **Fix**: Group ports by use case or capability. Apply Interface Segregation Principle.

### 3. Over-Engineering for Simple Apps
❌ **Mistake**: Full hexagonal architecture for a 200-line script.
✅ **Fix**: Use hexagonal when you genuinely need to swap adapters or test domain in isolation. For simple CRUD, layered architecture is fine.

### 4. Forgetting the Dependency Rule
❌ **Mistake**: Domain service imports from adapter package.
✅ **Fix**: Use compile-time module boundaries (separate Maven modules, Python packages) to enforce the rule.

### 5. Adapter Logic Leaking into Domain
❌ **Mistake**: Domain code handles database-specific concerns (connection management, retry logic).
✅ **Fix**: All infrastructure concerns belong in adapters. Domain just calls the port interface.

---

## When to Use / When NOT to Use

### ✅ Use Hexagonal Architecture When:

| Criteria | Why |
|---|---|
| **Complex business domain** | Keep business rules pure and clear |
| **Need to swap infrastructure** | Change DB, payment provider, messaging |
| **Testability is critical** | Test domain with zero external deps |
| **Long-lived application** | Frameworks change, domain logic stays |
| **Multiple entry points** | Same logic via REST, CLI, events, scheduled jobs |
| **Multiple output targets** | Same logic writes to DB, cache, search index |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Simple CRUD** | Massive over-engineering |
| **Prototyping / MVP** | Slows down initial development |
| **Framework does most of the work** | If 90% is framework glue, hexagonal adds no value |
| **Small team unfamiliar with pattern** | Learning curve can slow delivery |
| **Throwaway code** | Won't live long enough to benefit |

---

## Key Takeaways

- 🔌 **Ports = interfaces defined by domain** — what the business logic NEEDS from the outside world.
- 🔧 **Adapters = implementations** — connect real technologies (Postgres, Stripe, Kafka) to ports.
- ⬅️ **Driving adapters trigger the app** (REST, CLI, events) | **Driven adapters are called by the app** (DB, APIs, email).
- 🎯 **The domain has ZERO external dependencies** — no framework imports, no DB libraries, pure business logic.
- 🧪 **Testing becomes trivial** — swap real adapters with fakes/mocks, test domain logic in milliseconds.
- 🔄 **Swap anything without touching domain** — change Postgres to Mongo, Stripe to PayPal, REST to gRPC.
- 📐 **Dependencies point INWARD** — adapters depend on ports, ports are defined by domain. Never the reverse.

---

## What's Next?

We've seen how to structure code for testability and flexibility. Now let's look at a radically different approach to infrastructure. In **Chapter 4.10: Serverless Architecture**, we'll explore how to build applications without managing ANY servers — letting the cloud provider handle all infrastructure scaling.
