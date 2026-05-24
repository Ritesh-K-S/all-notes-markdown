# Event Sourcing — Storing Every Change as an Event

> **What you'll learn**: How event sourcing stores state as a sequence of immutable events instead of mutable rows, giving you a complete audit trail, time-travel capability, and the ability to rebuild any state.

---

## Real-Life Analogy

Think of your **bank account statement**:

Your bank doesn't store "current balance = $5,000" and call it a day. Instead, it stores **every transaction that ever happened**:

```
2024-01-01  Opening Balance          +$1,000   Balance: $1,000
2024-01-05  Salary Deposit           +$5,000   Balance: $6,000
2024-01-10  Rent Payment             -$1,500   Balance: $4,500
2024-01-12  Grocery Purchase         -$200     Balance: $4,300
2024-01-15  Freelance Payment        +$700     Balance: $5,000
```

Your current balance ($5,000) is **calculated by replaying all events** from the beginning. If there's ever a dispute, you can look at the full history. If you want to know your balance on Jan 12th, just replay events up to that date.

That's **event sourcing** — instead of storing "what the current state IS," you store "everything that happened" and derive the current state from those events.

---

## Core Concept Explained Step-by-Step

### Traditional State Storage vs Event Sourcing

```
TRADITIONAL (Store current state):

┌─────────────────────────────────────────┐
│           orders table                  │
├────────┬──────────┬────────┬────────────┤
│   id   │  status  │ total  │  updated   │
├────────┼──────────┼────────┼────────────┤
│ ORD-1  │ SHIPPED  │ $99.99 │ 2024-01-15 │ ← Only CURRENT state
│ ORD-2  │ CANCELLED│ $45.00 │ 2024-01-14 │ ← History is LOST
└────────┴──────────┴────────┴────────────┘

Questions you CAN'T answer:
- When was ORD-1 placed? When was it paid? When shipped?
- Was the price changed after placement?
- Who cancelled ORD-2 and why?


EVENT SOURCING (Store all events):

┌──────────────────────────────────────────────────────────────────┐
│                    order_events stream                            │
├──────────┬────────────────────┬────────────────────┬─────────────┤
│ event_id │     event_type     │       data         │  timestamp  │
├──────────┼────────────────────┼────────────────────┼─────────────┤
│    1     │ OrderPlaced        │ {items, total:$99} │ Jan 10 9:00 │
│    2     │ PaymentReceived    │ {txn_id, $99.99}   │ Jan 10 9:02 │
│    3     │ OrderConfirmed     │ {confirmed_by}     │ Jan 10 9:03 │
│    4     │ ItemShipped        │ {tracking_num}     │ Jan 13 14:00│
│    5     │ DeliveryConfirmed  │ {signed_by}        │ Jan 15 10:30│
└──────────┴────────────────────┴────────────────────┴─────────────┘

Current state = replay events 1→5 = Status: DELIVERED
State on Jan 11 = replay events 1→3 = Status: CONFIRMED
FULL HISTORY preserved forever!
```

### How Current State is Derived

```
EVENT STREAM for Order "ORD-1":

Event 1: OrderPlaced       → state = { status: "PLACED", total: $99.99, items: [...] }
Event 2: PaymentReceived   → state = { status: "PAID", total: $99.99, payment_id: "txn-1" }
Event 3: OrderConfirmed    → state = { status: "CONFIRMED", ... }
Event 4: ItemShipped       → state = { status: "SHIPPED", tracking: "TRK-123" }
Event 5: DeliveryConfirmed → state = { status: "DELIVERED", delivered_at: "..." }

     ┌───┐    ┌───┐    ┌───┐    ┌───┐    ┌───┐
     │ 1 │───▶│ 2 │───▶│ 3 │───▶│ 4 │───▶│ 5 │  = Current State
     └───┘    └───┘    └───┘    └───┘    └───┘
     
     ┌───┐    ┌───┐    ┌───┐
     │ 1 │───▶│ 2 │───▶│ 3 │  = State on Jan 11 (time travel!)
     └───┘    └───┘    └───┘
```

### The Event Store

An **event store** is an append-only database optimized for storing events:

```
┌────────────────────────────────────────────────────────────┐
│                     EVENT STORE                             │
│                                                            │
│  Stream: "Order-ORD-001"                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ [v1: OrderPlaced] [v2: Paid] [v3: Shipped] [v4: ...]│  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Stream: "Order-ORD-002"                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ [v1: OrderPlaced] [v2: Cancelled]                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Stream: "User-USR-042"                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ [v1: Registered] [v2: EmailChanged] [v3: Upgraded]   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  RULES:                                                    │
│  - Append-only (NEVER delete or modify events)             │
│  - Each stream represents one entity (aggregate)           │
│  - Events are ordered within a stream                      │
│  - Global ordering across streams (for projections)        │
└────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Aggregate + Events Pattern

```
┌─────────────────────────────────────────────────────────┐
│                    ORDER AGGREGATE                       │
│                                                         │
│  class Order:                                           │
│                                                         │
│    def place(customer_id, items):                        │
│        # Validate business rules                        │
│        # Emit event (don't mutate state directly!)      │
│        emit(OrderPlaced(customer_id, items, total))     │
│                                                         │
│    def pay(payment_id, amount):                          │
│        if self.status != "PLACED":                      │
│            raise InvalidStateError                      │
│        emit(PaymentReceived(payment_id, amount))        │
│                                                         │
│    def apply(event):                                    │
│        # Mutate state ONLY through event application    │
│        match event:                                     │
│          OrderPlaced → self.status = "PLACED"           │
│          PaymentReceived → self.status = "PAID"         │
│          ...                                            │
│                                                         │
│  RULE: State changes ONLY through events.               │
│        No direct mutations allowed.                     │
└─────────────────────────────────────────────────────────┘
```

### Loading an Aggregate (Replay)

```
REQUEST: "Get Order ORD-001"

1. Load all events for stream "Order-ORD-001":
   [OrderPlaced, PaymentReceived, OrderConfirmed, ItemShipped]

2. Create empty Order object

3. Apply each event in order:
   apply(OrderPlaced)     → status=PLACED, total=$99
   apply(PaymentReceived) → status=PAID, payment_ref="txn-1"
   apply(OrderConfirmed)  → status=CONFIRMED
   apply(ItemShipped)     → status=SHIPPED, tracking="TRK-123"

4. Current state is now fully reconstructed!
```

### Snapshots (Performance Optimization)

For aggregates with thousands of events, replaying from scratch is slow. Use **snapshots**:

```
WITHOUT SNAPSHOTS:
Replay 10,000 events every time = SLOW

WITH SNAPSHOTS:
┌──────────────────────────────────────────────────────────┐
│  Events: [e1][e2][e3]...[e5000]  [SNAPSHOT]  [e5001]...[e5010]  │
│                                      │                    │
│  Load snapshot (state at event 5000) │                    │
│  Then replay only 10 events          │                    │
│  Result: Same state, much faster!    │                    │
└──────────────────────────────────────────────────────────┘

Snapshot = serialized state at a point in time
Take a new snapshot every N events (e.g., every 100)
```

### Event Sourcing + CQRS (Perfect Combination)

```
┌──────────┐   Command    ┌──────────────┐    Events    ┌─────────────┐
│  Client  │─────────────▶│   Aggregate  │─────────────▶│ Event Store │
│ (writes) │              │   (Domain)   │              │ (append-only│
└──────────┘              └──────────────┘              └──────┬──────┘
                                                               │
                                                    publish to event bus
                                                               │
                                                               ▼
                                                        ┌─────────────┐
                                                        │    Kafka    │
                                                        └──────┬──────┘
                                                               │
                                               ┌───────────────┼──────────────┐
                                               ▼               ▼              ▼
                                        ┌────────────┐  ┌────────────┐  ┌──────────┐
                                        │ Read Model │  │ Read Model │  │ Read     │
                                        │ (Postgres  │  │(Elastic-   │  │ Model    │
                                        │  summary)  │  │ search)    │  │ (Redis)  │
                                        └────────────┘  └────────────┘  └──────────┘
                                               ▲               ▲              ▲
                                               │               │              │
┌──────────┐   Query     ┌──────────────┐     │               │              │
│  Client  │────────────▶│ Query Handler │─────┘───────────────┘──────────────┘
│ (reads)  │◀────────────│              │
└──────────┘   Results   └──────────────┘
```

---

## Code Examples

### Python (Event Sourcing Implementation)

```python
# domain/events.py — Domain events (immutable facts)
from dataclasses import dataclass, field
from datetime import datetime
import uuid

@dataclass(frozen=True)  # Immutable!
class OrderPlaced:
    order_id: str
    customer_id: str
    items: list
    total: float
    occurred_at: datetime = field(default_factory=datetime.utcnow)
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))

@dataclass(frozen=True)
class PaymentReceived:
    order_id: str
    payment_id: str
    amount: float
    occurred_at: datetime = field(default_factory=datetime.utcnow)

@dataclass(frozen=True)
class OrderShipped:
    order_id: str
    tracking_number: str
    carrier: str
    occurred_at: datetime = field(default_factory=datetime.utcnow)

# domain/order.py — Aggregate that produces and applies events
class Order:
    def __init__(self):
        self.id = None
        self.status = None
        self.items = []
        self.total = 0
        self._pending_events = []  # Events not yet persisted
    
    # --- COMMANDS (produce events) ---
    @classmethod
    def place(cls, customer_id: str, items: list):
        """Create a new order (command)."""
        order = cls()
        total = sum(item['price'] * item['quantity'] for item in items)
        event = OrderPlaced(
            order_id=str(uuid.uuid4()),
            customer_id=customer_id,
            items=items,
            total=total
        )
        order._apply(event)
        order._pending_events.append(event)
        return order
    
    def pay(self, payment_id: str, amount: float):
        """Record payment (command)."""
        if self.status != "PLACED":
            raise InvalidStateError(f"Cannot pay order in status {self.status}")
        event = PaymentReceived(self.id, payment_id, amount)
        self._apply(event)
        self._pending_events.append(event)
    
    def ship(self, tracking_number: str, carrier: str):
        """Ship order (command)."""
        if self.status != "PAID":
            raise InvalidStateError("Cannot ship unpaid order")
        event = OrderShipped(self.id, tracking_number, carrier)
        self._apply(event)
        self._pending_events.append(event)
    
    # --- EVENT APPLICATION (mutate state) ---
    def _apply(self, event):
        """Apply event to update internal state."""
        if isinstance(event, OrderPlaced):
            self.id = event.order_id
            self.status = "PLACED"
            self.items = event.items
            self.total = event.total
        elif isinstance(event, PaymentReceived):
            self.status = "PAID"
        elif isinstance(event, OrderShipped):
            self.status = "SHIPPED"
            self.tracking = event.tracking_number
    
    # --- RECONSTRUCTION ---
    @classmethod
    def from_events(cls, events: list):
        """Rebuild aggregate from event history (replay)."""
        order = cls()
        for event in events:
            order._apply(event)
        return order

# infrastructure/event_store.py — Append-only event storage
class EventStore:
    def __init__(self, db):
        self.db = db
    
    def save(self, stream_id: str, events: list, expected_version: int):
        """Save events with optimistic concurrency check."""
        current_version = self._get_stream_version(stream_id)
        if current_version != expected_version:
            raise ConcurrencyError("Stream was modified by another process")
        
        for i, event in enumerate(events):
            self.db.execute(
                "INSERT INTO events (stream_id, version, event_type, data, timestamp) "
                "VALUES (%s, %s, %s, %s, %s)",
                (stream_id, expected_version + i + 1,
                 type(event).__name__, serialize(event), event.occurred_at)
            )
    
    def load(self, stream_id: str) -> list:
        """Load all events for a stream."""
        rows = self.db.query(
            "SELECT event_type, data FROM events "
            "WHERE stream_id = %s ORDER BY version ASC",
            (stream_id,)
        )
        return [deserialize(row['event_type'], row['data']) for row in rows]
```

### Java (Event Sourcing with Axon Framework style)

```java
// Order.java — Event-sourced aggregate
public class Order {
    private String orderId;
    private String status;
    private BigDecimal total;
    private List<DomainEvent> pendingEvents = new ArrayList<>();
    
    // --- COMMAND HANDLERS (produce events) ---
    public static Order place(String customerId, List<OrderItem> items) {
        Order order = new Order();
        BigDecimal total = items.stream()
            .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        order.apply(new OrderPlaced(UUID.randomUUID().toString(), 
                                     customerId, items, total));
        return order;
    }
    
    public void pay(String paymentId, BigDecimal amount) {
        if (!"PLACED".equals(this.status)) {
            throw new IllegalStateException("Cannot pay order in status: " + status);
        }
        apply(new PaymentReceived(this.orderId, paymentId, amount));
    }
    
    public void ship(String trackingNumber) {
        if (!"PAID".equals(this.status)) {
            throw new IllegalStateException("Cannot ship unpaid order");
        }
        apply(new OrderShipped(this.orderId, trackingNumber));
    }
    
    // --- EVENT HANDLERS (mutate state) ---
    private void apply(DomainEvent event) {
        mutate(event);
        pendingEvents.add(event);
    }
    
    private void mutate(DomainEvent event) {
        if (event instanceof OrderPlaced e) {
            this.orderId = e.getOrderId();
            this.status = "PLACED";
            this.total = e.getTotal();
        } else if (event instanceof PaymentReceived e) {
            this.status = "PAID";
        } else if (event instanceof OrderShipped e) {
            this.status = "SHIPPED";
        }
    }
    
    // --- REBUILD FROM HISTORY ---
    public static Order fromHistory(List<DomainEvent> events) {
        Order order = new Order();
        events.forEach(order::mutate);
        return order;
    }
    
    public List<DomainEvent> getPendingEvents() {
        return Collections.unmodifiableList(pendingEvents);
    }
}
```

---

## Infrastructure Example

### Event Store Database Schema

```sql
-- PostgreSQL event store schema
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    stream_id       VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    data            JSONB NOT NULL,
    metadata        JSONB DEFAULT '{}',
    timestamp       TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Optimistic concurrency: no two events with same stream+version
    UNIQUE (stream_id, version)
);

-- Index for loading stream events
CREATE INDEX idx_events_stream ON events (stream_id, version);

-- Index for global ordering (for projections)
CREATE INDEX idx_events_id ON events (id);

-- Snapshots table (optional optimization)
CREATE TABLE snapshots (
    stream_id       VARCHAR(255) PRIMARY KEY,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,
    timestamp       TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### EventStoreDB (Purpose-built event store)

```yaml
# docker-compose.yml — EventStoreDB cluster
version: '3.8'
services:
  eventstore:
    image: eventstore/eventstore:latest
    ports:
      - "2113:2113"  # HTTP API
      - "1113:1113"  # TCP API
    environment:
      EVENTSTORE_CLUSTER_SIZE: 3
      EVENTSTORE_RUN_PROJECTIONS: All
      EVENTSTORE_INSECURE: true
    volumes:
      - eventstore-data:/var/lib/eventstore
```

---

## Real-World Example

### Banking & Financial Systems

Every bank uses event sourcing for **ledgers** — they NEVER overwrite a balance:

```
ACCOUNT LEDGER (event sourced):

Stream: "Account-ACC-001"
┌────┬──────────────────────┬──────────────┬─────────────────┐
│ v# │      Event           │    Amount    │  Running Bal    │
├────┼──────────────────────┼──────────────┼─────────────────┤
│  1 │ AccountOpened        │      $0      │      $0         │
│  2 │ DepositMade          │  +$5,000     │   $5,000        │
│  3 │ WithdrawalMade       │  -$1,500     │   $3,500        │
│  4 │ TransferReceived     │    +$200     │   $3,700        │
│  5 │ FeesCharged          │     -$15     │   $3,685        │
│  6 │ InterestAccrued      │     +$12     │   $3,697        │
└────┴──────────────────────┴──────────────┴─────────────────┘

Current balance = replay all events = $3,697
Balance on v3 = replay events 1-3 = $3,500
```

### Git — Version Control is Event Sourcing

**Git** is event sourcing for code:
- Each **commit** is an event (immutable record of what changed)
- Current state = replay all commits from initial commit
- You can "time travel" to any previous commit
- You never delete history (only append)

### Walmart — Inventory Events

```
Stream: "Inventory-SKU-12345"

[v1: ItemReceived(qty: 1000)]
[v2: ItemSold(qty: 1)]
[v3: ItemSold(qty: 3)]
[v4: ItemReceived(qty: 500)]
[v5: ItemDamaged(qty: 2)]
[v6: ItemReturned(qty: 1)]
...

Current stock = 1000 - 1 - 3 + 500 - 2 + 1 = 1495
Stock at v3 = 1000 - 1 - 3 = 996
```

---

## Common Mistakes / Pitfalls

### 1. Modifying Past Events
❌ **Mistake**: Going back and "fixing" a wrong event by editing it.
✅ **Fix**: Events are IMMUTABLE. To correct a mistake, add a **compensating event** (e.g., `OrderCorrected` or `RefundIssued`).

### 2. Huge Events
❌ **Mistake**: Storing entire entity state in every event.
✅ **Fix**: Events should contain only what CHANGED, not the full state.

```
BAD: OrderUpdated { order: { all 50 fields... } }  // What changed??
GOOD: ShippingAddressChanged { order_id: "x", new_address: "..." }
```

### 3. Not Using Snapshots for Long Streams
❌ **Mistake**: Aggregate has 100,000 events and you replay all of them on every request.
✅ **Fix**: Take periodic snapshots. Load snapshot + replay only events after it.

### 4. Coupling Events to Internal Structure
❌ **Mistake**: Event names/data match internal implementation details.
✅ **Fix**: Events should represent **domain concepts** that are stable over time.

### 5. No Event Versioning Strategy
❌ **Mistake**: Changing event schema breaks replay of old events.
✅ **Fix**: Use **upcasters** — transform old event formats to new ones during replay.

```
Version 1 event: { "customer_name": "Alice Smith" }
Version 2 event: { "first_name": "Alice", "last_name": "Smith" }

Upcaster: v1 → split customer_name into first_name + last_name
```

---

## When to Use / When NOT to Use

### ✅ Use Event Sourcing When:

| Criteria | Why |
|---|---|
| **Complete audit trail required** | Financial systems, compliance, legal |
| **Need to answer "what happened"** | Debugging, analytics, dispute resolution |
| **Complex business processes** | Track every state transition |
| **Temporal queries needed** | "What was the state on March 15?" |
| **Event-driven architecture in place** | Natural fit with EDA + CQRS |
| **Need to rebuild/derive new views** | Replay events into new read models |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Simple CRUD** | Massive over-engineering |
| **No audit requirements** | Standard DB with soft-deletes is simpler |
| **Team unfamiliar with pattern** | Steep learning curve |
| **Need to delete data (GDPR)** | Event immutability conflicts with "right to be forgotten" |
| **Simple current-state queries** | No need for history |

---

## Key Takeaways

- 📜 **Event sourcing stores state as a sequence of immutable events** — you never store "current state" directly.
- 🔄 **Current state = replay all events** — apply events in order to reconstruct the aggregate.
- 🕰️ **Time travel is built-in** — replay events up to any point to see historical state.
- 📷 **Snapshots prevent replay performance issues** — periodically save state to avoid replaying thousands of events.
- 🏦 **Banks and financial systems use this natively** — ledgers are event-sourced by nature.
- ✏️ **Events are IMMUTABLE** — never modify past events. Use compensating events for corrections.
- 🤝 **CQRS + Event Sourcing = powerful combination** — events feed projections that build optimized read models.

---

## What's Next?

Now let's look at a pattern that focuses on making your application's core logic independent of frameworks and infrastructure. In **Chapter 4.9: Hexagonal Architecture (Ports & Adapters)**, we'll learn how to structure code so that your business logic doesn't depend on databases, web frameworks, or any external technology.
