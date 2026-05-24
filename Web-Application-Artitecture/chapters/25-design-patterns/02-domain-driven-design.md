# Domain-Driven Design (DDD) Essentials

> **What you'll learn**: How to structure complex business applications by modeling your code around the business domain — the methodology used by teams at Netflix, Uber, and Amazon to keep million-line codebases organized and maintainable.

---

## Real-Life Analogy

Imagine you're building a hospital management system. You sit with doctors, nurses, billing staff, and lab technicians. You quickly notice something fascinating:

- When a **doctor** says "patient record," they mean diagnosis history and treatment plans.
- When a **billing staff** says "patient record," they mean insurance info and payment history.
- When a **lab technician** says "patient record," they mean blood samples and test results.

Same word. **Completely different meanings depending on who's talking.**

Domain-Driven Design says: **Don't force everyone to share one definition.** Let each department (bounded context) have its own clear model. Connect them through well-defined interfaces.

It's like how the same person is "Mom" at home, "Dr. Singh" at the hospital, and "Customer #4521" at the bank. Same entity, different contexts, different relevant attributes.

---

## What is Domain-Driven Design?

DDD is a set of principles for **organizing complex business software** so that:
1. The code structure mirrors the business structure
2. Developers and business experts speak the same language
3. Complexity is managed through clear boundaries
4. The system can evolve as the business evolves

```
Traditional Approach:                DDD Approach:
─────────────────────               ──────────────────────
Code organized by                   Code organized by
TECHNICAL layers:                   BUSINESS capabilities:

├── controllers/                    ├── ordering/
│   ├── UserController              │   ├── domain/
│   ├── OrderController             │   ├── application/
│   └── ProductController           │   └── infrastructure/
├── services/                       ├── inventory/
│   ├── UserService                 │   ├── domain/
│   ├── OrderService                │   ├── application/
│   └── ProductService              │   └── infrastructure/
├── repositories/                   ├── shipping/
│   ├── UserRepo                    │   ├── domain/
│   ├── OrderRepo                   │   ├── application/
│   └── ProductRepo                 │   └── infrastructure/
└── models/                         └── billing/
    ├── User                            ├── domain/
    ├── Order                           ├── application/
    └── Product                         └── infrastructure/
```

---

## Core Concepts — The Building Blocks of DDD

### 1. Ubiquitous Language

The MOST important concept in DDD. It means: **developers and business people must use the SAME words for the same things.**

```
BAD (Technical jargon):
─────────────────────────────────────────────
Developer: "The UserEntity's statusFlag gets toggled 
            when the cron job runs the batch processor."
Business:  "...What?"

GOOD (Ubiquitous Language):
─────────────────────────────────────────────
Everyone:  "When a subscription expires, the member 
            account is deactivated automatically."
```

If the business says "Order," your code has a class called `Order` — not `PurchaseRequest`, not `TransactionEntity`. The code reads like the business speaks.

---

### 2. Bounded Context

A **Bounded Context** is a boundary within which a particular model and language applies. Different contexts can have different models for the same real-world thing.

```
┌─────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE SYSTEM                              │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │  ORDERING Context │  │ INVENTORY Context│  │SHIPPING Context│  │
│  │                    │  │                  │  │               │  │
│  │  Order             │  │  Product         │  │  Shipment     │  │
│  │  - items[]         │  │  - sku           │  │  - tracking#  │  │
│  │  - totalPrice      │  │  - stockLevel    │  │  - carrier    │  │
│  │  - customer        │  │  - warehouse     │  │  - address    │  │
│  │  - status          │  │  - reorderPoint  │  │  - weight     │  │
│  │                    │  │                  │  │               │  │
│  │  "Product" here =  │  │  "Product" here =│  │  "Product" =  │  │
│  │  name + price +    │  │  SKU + quantity + │  │  dimensions + │  │
│  │  image             │  │  location        │  │  weight       │  │
│  └──────────────────┘  └──────────────────┘  └───────────────┘  │
│           │                      │                     │         │
│           └──────────────────────┴─────────────────────┘         │
│                    Connected via Events / APIs                    │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight**: "Product" means something different in each context. That's fine! Don't force a single `Product` model to serve everyone.

---

### 3. Entities

Objects defined by their **identity**, not their attributes. Two orders with the same items are still different orders because they have different IDs.

```python
# Entity - identity matters
class Order:
    def __init__(self, order_id: str):
        self.order_id = order_id  # THIS defines the entity
        self.items = []
        self.status = "CREATED"
    
    def __eq__(self, other):
        return self.order_id == other.order_id  # Equality by ID!
```

---

### 4. Value Objects

Objects defined by their **attributes**, not an identity. Two `Money(100, "USD")` instances are identical and interchangeable.

```python
# Value Object - attributes define equality
from dataclasses import dataclass

@dataclass(frozen=True)  # Immutable!
class Money:
    amount: float
    currency: str
    
    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

# Two Money objects with same values ARE the same
Money(100, "USD") == Money(100, "USD")  # True!
```

---

### 5. Aggregates

A cluster of entities and value objects treated as a **single unit** for data changes. The **Aggregate Root** is the only entry point.

```
┌─────────────────────────────────────────┐
│          ORDER AGGREGATE                 │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │     Order (Aggregate Root)        │   │
│  │     - orderId                     │   │
│  │     - status                      │   │
│  │     - placedAt                    │   │
│  └──────────────┬───────────────────┘   │
│                 │ contains                │
│    ┌────────────┴────────────┐           │
│    ▼                         ▼           │
│  ┌──────────────┐  ┌──────────────┐     │
│  │  OrderItem   │  │  OrderItem   │     │
│  │  - productId │  │  - productId │     │
│  │  - quantity  │  │  - quantity  │     │
│  │  - price     │  │  - price     │     │
│  └──────────────┘  └──────────────┘     │
│                                          │
│  RULES:                                  │
│  • Access items ONLY through Order       │
│  • Save/Load the entire aggregate        │
│  • One transaction per aggregate         │
└─────────────────────────────────────────┘
```

**Rules of Aggregates**:
- Only the root can be referenced from outside
- All changes go through the root
- One transaction = one aggregate (no multi-aggregate transactions)
- Keep aggregates small!

---

### 6. Domain Events

Something meaningful that happened in the domain. Events are named in **past tense** because they represent facts that already occurred.

```
OrderPlaced       ← An order was placed
PaymentReceived   ← Payment was received
ItemShipped       ← Item was shipped
SubscriptionExpired ← A subscription expired
```

```
┌──────────┐  OrderPlaced   ┌────────────┐
│ Ordering │───────────────▶│ Inventory  │
│ Context  │                │ Context    │
└──────────┘                └────────────┘
                                  │
                            ReserveStock
                                  │
                                  ▼
                            ┌────────────┐
                            │  Shipping  │
                            │  Context   │
                            └────────────┘
```

---

### 7. Repositories

The interface through which you load and save aggregates. It hides the database details from the domain.

```
┌──────────────────────────────────────────────────┐
│                  DOMAIN LAYER                      │
│                                                    │
│  OrderService uses OrderRepository (interface)     │
│  - findById(orderId) → Order                      │
│  - save(order) → void                             │
│                                                    │
└────────────────────────┬─────────────────────────┘
                         │ implements
                         ▼
┌──────────────────────────────────────────────────┐
│              INFRASTRUCTURE LAYER                  │
│                                                    │
│  PostgresOrderRepository implements               │
│  OrderRepository                                  │
│  - Uses SQL to load/save Order aggregates         │
│                                                    │
└──────────────────────────────────────────────────┘
```

---

### 8. Domain Services

Business logic that doesn't naturally belong to any single entity. For example, "transfer money between accounts" involves two accounts — it doesn't belong to either one.

```python
class MoneyTransferService:
    """Domain Service - logic that spans multiple aggregates"""
    
    def transfer(self, source: Account, target: Account, amount: Money):
        if not source.has_sufficient_funds(amount):
            raise InsufficientFundsError()
        
        source.debit(amount)
        target.credit(amount)
        
        return MoneyTransferred(source.id, target.id, amount)
```

---

## How It Works Internally — The Layered Architecture of DDD

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                         │
│            (REST Controllers, GraphQL, CLI)                   │
│                                                               │
│  Receives requests, calls Application Layer, returns response │
└────────────────────────────────┬────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                          │
│             (Use Cases / Application Services)                │
│                                                               │
│  Orchestrates domain objects to fulfill a use case            │
│  NO business logic here — just coordination                  │
│  Handles transactions, event publishing                       │
└────────────────────────────────┬────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│                      DOMAIN LAYER                             │
│        (Entities, Value Objects, Aggregates, Services)        │
│                                                               │
│  ALL business rules live here                                 │
│  No dependency on frameworks, databases, or UI               │
│  Pure business logic — testable without infrastructure        │
└────────────────────────────────┬────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│                  INFRASTRUCTURE LAYER                         │
│     (Database, Message Queue, External APIs, File System)    │
│                                                               │
│  Implements interfaces defined in the Domain layer           │
│  Actual PostgreSQL queries, Kafka producers, HTTP calls       │
└─────────────────────────────────────────────────────────────┘
```

**Key principle**: Dependencies point INWARD. The Domain layer knows nothing about databases, HTTP, or frameworks. It's pure business logic.

---

## Code Examples

### Python: Full DDD Structure for an Order System

```python
# ══════════════════════════════════════════════════════
# DOMAIN LAYER - Pure business logic, no dependencies
# ══════════════════════════════════════════════════════

# domain/value_objects.py
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: float
    currency: str = "USD"
    
    def add(self, other: "Money") -> "Money":
        assert self.currency == other.currency
        return Money(self.amount + other.amount, self.currency)
    
    def multiply(self, factor: int) -> "Money":
        return Money(self.amount * factor, self.currency)

@dataclass(frozen=True)
class Address:
    street: str
    city: str
    zipcode: str
    country: str

# domain/entities.py
class OrderItem:
    def __init__(self, product_id: str, product_name: str, 
                 price: Money, quantity: int):
        if quantity <= 0:
            raise ValueError("Quantity must be positive")
        self.product_id = product_id
        self.product_name = product_name
        self.price = price
        self.quantity = quantity
    
    @property
    def subtotal(self) -> Money:
        return self.price.multiply(self.quantity)

# domain/aggregates.py
class Order:
    """Aggregate Root - all access goes through here"""
    
    def __init__(self, order_id: str, customer_id: str):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items: list[OrderItem] = []
        self.status = "DRAFT"
        self._events: list = []  # Domain events
    
    def add_item(self, product_id: str, name: str, price: Money, qty: int):
        if self.status != "DRAFT":
            raise InvalidOperationError("Cannot modify a placed order")
        item = OrderItem(product_id, name, price, qty)
        self.items.append(item)
    
    def place(self):
        """Place the order - business rule enforcement"""
        if not self.items:
            raise InvalidOperationError("Cannot place empty order")
        if self.total.amount > 10000:
            raise OrderLimitExceeded("Order exceeds $10,000 limit")
        
        self.status = "PLACED"
        # Raise domain event
        self._events.append(OrderPlaced(
            order_id=self.order_id,
            customer_id=self.customer_id,
            total=self.total
        ))
    
    @property
    def total(self) -> Money:
        if not self.items:
            return Money(0)
        result = Money(0)
        for item in self.items:
            result = result.add(item.subtotal)
        return result

# domain/events.py
@dataclass
class OrderPlaced:
    order_id: str
    customer_id: str
    total: Money

# domain/repository.py (Interface only!)
from abc import ABC, abstractmethod

class OrderRepository(ABC):
    @abstractmethod
    def find_by_id(self, order_id: str) -> Order | None:
        pass
    
    @abstractmethod
    def save(self, order: Order) -> None:
        pass
```

```python
# ══════════════════════════════════════════════════════
# APPLICATION LAYER - Orchestrates use cases
# ══════════════════════════════════════════════════════

# application/place_order.py
class PlaceOrderCommand:
    def __init__(self, order_id: str):
        self.order_id = order_id

class PlaceOrderHandler:
    def __init__(self, order_repo: OrderRepository, 
                 event_publisher: EventPublisher):
        self.order_repo = order_repo
        self.event_publisher = event_publisher
    
    def handle(self, command: PlaceOrderCommand):
        # Load aggregate
        order = self.order_repo.find_by_id(command.order_id)
        if not order:
            raise OrderNotFound(command.order_id)
        
        # Execute domain logic (business rules enforced inside)
        order.place()
        
        # Persist
        self.order_repo.save(order)
        
        # Publish domain events
        for event in order._events:
            self.event_publisher.publish(event)
```

### Java: Full DDD Structure

```java
// ══════════════════════════════════════════════════════
// DOMAIN LAYER
// ══════════════════════════════════════════════════════

// domain/valueobject/Money.java
public record Money(BigDecimal amount, String currency) {
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount cannot be negative");
    }
    
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), currency);
    }
    
    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
    }
}

// domain/aggregate/Order.java
public class Order {
    private final String orderId;
    private final String customerId;
    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status = OrderStatus.DRAFT;
    private final List<DomainEvent> events = new ArrayList<>();
    
    public Order(String orderId, String customerId) {
        this.orderId = orderId;
        this.customerId = customerId;
    }
    
    public void addItem(String productId, String name, Money price, int qty) {
        if (status != OrderStatus.DRAFT)
            throw new InvalidOperationException("Cannot modify placed order");
        items.add(new OrderItem(productId, name, price, qty));
    }
    
    public void place() {
        if (items.isEmpty())
            throw new InvalidOperationException("Cannot place empty order");
        
        Money total = calculateTotal();
        if (total.amount().compareTo(new BigDecimal("10000")) > 0)
            throw new OrderLimitExceededException("Exceeds $10,000");
        
        this.status = OrderStatus.PLACED;
        events.add(new OrderPlaced(orderId, customerId, total));
    }
    
    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(new Money(BigDecimal.ZERO, "USD"), Money::add);
    }
    
    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(events);
    }
}

// domain/repository/OrderRepository.java (Interface!)
public interface OrderRepository {
    Optional<Order> findById(String orderId);
    void save(Order order);
}
```

```java
// ══════════════════════════════════════════════════════
// APPLICATION LAYER
// ══════════════════════════════════════════════════════

// application/PlaceOrderHandler.java
@Service
@Transactional
public class PlaceOrderHandler {
    private final OrderRepository orderRepo;
    private final EventPublisher eventPublisher;
    
    public PlaceOrderHandler(OrderRepository orderRepo, 
                             EventPublisher eventPublisher) {
        this.orderRepo = orderRepo;
        this.eventPublisher = eventPublisher;
    }
    
    public void handle(PlaceOrderCommand command) {
        Order order = orderRepo.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));
        
        order.place();  // Domain logic happens here
        
        orderRepo.save(order);
        
        order.getDomainEvents().forEach(eventPublisher::publish);
    }
}
```

---

## Context Mapping — How Bounded Contexts Relate

```
┌─────────────────────────────────────────────────────────────────┐
│                     CONTEXT MAP                                   │
│                                                                   │
│  ┌──────────┐     Shared        ┌──────────┐                    │
│  │ Identity │     Kernel         │  Billing │                    │
│  │ Context  │◄──────────────────▶│  Context │                    │
│  └────┬─────┘                    └────┬─────┘                    │
│       │                               │                          │
│       │ Customer/                      │                          │
│       │ Supplier                       │ Conformist              │
│       │                               │                          │
│       ▼                               ▼                          │
│  ┌──────────┐     Published     ┌──────────┐                    │
│  │ Ordering │     Language       │ Payments │                    │
│  │ Context  │────────────────────│ Context  │                    │
│  └────┬─────┘    (Events)       └──────────┘                    │
│       │                                                          │
│       │ Anti-Corruption                                          │
│       │ Layer (ACL)                                              │
│       ▼                                                          │
│  ┌──────────┐                                                    │
│  │ Legacy   │  ← Old system we can't modify                     │
│  │ Inventory│                                                    │
│  └──────────┘                                                    │
│                                                                   │
│  RELATIONSHIP TYPES:                                             │
│  • Shared Kernel: Both contexts share some code                  │
│  • Customer/Supplier: One depends on the other                   │
│  • Conformist: Downstream conforms to upstream's model           │
│  • Anti-Corruption Layer: Translator protects from bad models    │
│  • Published Language: Context publishes events/APIs             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Amazon E-Commerce

Amazon's codebase is organized by bounded contexts that align with their business:

| Bounded Context | Responsibility | Key Entities |
|----------------|---------------|--------------|
| Catalog | Product information, search | Product, Category, Review |
| Ordering | Cart, checkout, order tracking | Order, Cart, OrderItem |
| Inventory | Stock levels, warehouses | StockUnit, Warehouse, Location |
| Shipping | Delivery, tracking | Shipment, Route, Carrier |
| Payments | Charge, refund | Payment, Refund, PaymentMethod |
| Identity | Users, authentication | Customer, Credentials |

Each context is a separate team with separate deployables. They communicate via events:

```
Customer places order:
  Ordering Context: "OrderPlaced" event
       │
       ├──▶ Inventory: Reserve stock
       ├──▶ Payments: Charge customer  
       ├──▶ Shipping: Prepare shipment
       └──▶ Notifications: Send email
```

### Uber

Uber uses DDD extensively:
- **Trip Context**: Ride requests, matching, pricing
- **Driver Context**: Driver profiles, availability, earnings
- **Payments Context**: Fare calculation, payment processing
- **Maps Context**: Routes, ETAs, geolocation

Each context has its own team, data store, and deployment pipeline.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Sharing a database between contexts | Tight coupling; one change breaks everything | Each context owns its data |
| Making aggregates too large | Slow performance, lock contention | Keep aggregates small (2-3 entities max) |
| Putting business logic in services instead of entities | Anemic domain model — entities become dumb data bags | Put logic WHERE THE DATA IS |
| Ignoring ubiquitous language | Developers and business speak different languages → bugs | Regular meetings to align on terms |
| Applying DDD to simple CRUD apps | Over-engineering; adds complexity without benefit | Only use DDD for complex domains |
| Cross-aggregate transactions | Violates aggregate boundaries, causes scaling issues | Use domain events for eventual consistency |

---

## When to Use / When NOT to Use

### Use DDD When:
- Domain is complex (many business rules, exceptions, edge cases)
- Business logic changes frequently
- Multiple teams work on the same system
- You're building microservices and need clear boundaries
- The cost of getting the model wrong is high (finance, healthcare)
- Domain experts are available for collaboration

### Do NOT Use DDD When:
- Simple CRUD applications (blog, basic todo app)
- Technical domains without complex business rules (ETL pipelines)
- Small teams with simple requirements
- Prototypes or MVPs (over-engineering kills speed)
- The domain is well-understood and stable

> **Rule of thumb**: If your biggest challenge is technical (how to store data, scale reads), DDD might be overkill. If your biggest challenge is business complexity (rules, exceptions, workflows), DDD is essential.

---

## Key Takeaways

1. **Ubiquitous Language is #1** — If you do nothing else from DDD, align your code vocabulary with your business vocabulary. This alone prevents 50% of miscommunication bugs.

2. **Bounded Contexts prevent model corruption** — Different parts of the system need different models of the same concept. Let them diverge; connect through events and APIs.

3. **Aggregates enforce consistency** — One transaction per aggregate. Keep them small. Use eventual consistency between aggregates.

4. **Domain Events enable loose coupling** — Instead of "Order service calls Inventory service," do "Order publishes OrderPlaced event; Inventory subscribes."

5. **The domain layer has ZERO infrastructure dependencies** — If your Order class imports SQL or HTTP libraries, you're doing it wrong.

6. **DDD is expensive** — It requires close collaboration with domain experts, skilled developers, and more upfront design. Only apply it where the complexity justifies the cost.

7. **Start with strategic DDD** (bounded contexts, context mapping) before tactical DDD (entities, value objects, aggregates). Getting boundaries right matters more than getting code patterns right.

---

## What's Next?

Next, we'll look at **API Design Best Practices** — how to design the interfaces between your bounded contexts (and for external consumers) following REST principles, pagination patterns, and error handling conventions.

See: [03-api-design-best-practices.md](./03-api-design-best-practices.md)
