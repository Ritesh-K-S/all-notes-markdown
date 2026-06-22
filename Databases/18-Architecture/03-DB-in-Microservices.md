# 🧩 Chapter 7.3 — Database in Microservices Architecture

> **Level:** 🔴 Advanced | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~5-6 hours
> **Prerequisites:** Chapter 7.1 (Sharding), Chapter 7.2 (Replication), Chapter 1.8 (Transactions)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **Database per Service** — the foundational microservices pattern
- Master the **Saga Pattern** — how to do transactions across services without 2PC
- Implement **Event Sourcing** — storing events instead of state
- Design with **CQRS** — separate read and write models
- Use the **Outbox Pattern** — reliable event publishing without dual-writes
- Handle **data consistency** in distributed systems without losing your mind
- Know which pattern to pick for **real-world scenarios** (e-commerce, banking, social)

---

## 🧠 The Big Shift — From Monolith to Microservices

### The Monolith Database

```
                        THE MONOLITH
                        ═══════════

            ┌──────────────────────────────┐
            │        Monolith App          │
            │                              │
            │  ┌────────┐ ┌────────┐       │
            │  │ Users  │ │ Orders │       │
            │  │ Module │ │ Module │       │
            │  └────┬───┘ └────┬───┘       │
            │       │          │            │
            │  ┌────┴──┐ ┌────┴──────┐     │
            │  │Payment│ │ Inventory │     │
            │  │ Module│ │ Module    │     │
            │  └────┬──┘ └────┬──────┘     │
            │       │          │            │
            └───────┼──────────┼────────────┘
                    │          │
                    ▼          ▼
            ┌──────────────────────────────┐
            │     ONE BIG DATABASE         │
            │                              │
            │  ┌───────┐ ┌────────┐        │
            │  │ users │ │ orders │        │
            │  └───────┘ └────────┘        │
            │  ┌──────────┐ ┌───────────┐  │
            │  │ payments │ │ inventory │  │
            │  └──────────┘ └───────────┘  │
            │                              │
            │  🔒 ACID Transactions        │
            │  🔗 JOINs work perfectly     │
            │  📐 Foreign keys enforced    │
            │                              │
            │  Life is simple. ☀️           │
            └──────────────────────────────┘
```

```sql
-- In a monolith, this transaction is EASY:
BEGIN TRANSACTION;

    -- 1. Create order
    INSERT INTO orders (user_id, total) VALUES (42, 299.99);
    
    -- 2. Charge payment
    INSERT INTO payments (order_id, amount, status) 
    VALUES (LAST_INSERT_ID(), 299.99, 'charged');
    
    -- 3. Reduce inventory
    UPDATE inventory SET quantity = quantity - 1 
    WHERE product_id = 101 AND quantity > 0;
    
    -- 4. All or nothing!
    IF @@ROWCOUNT = 0 THEN
        ROLLBACK;  -- Out of stock!
    ELSE
        COMMIT;    -- Everything succeeded!
    END IF;

-- ONE database, ONE transaction, ACID guarantees. 
```

### The Microservices Database Problem

```
                    THE MICROSERVICES WORLD
                    ═══════════════════════

   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
   │  User    │    │  Order   │    │  Payment │    │ Inventory│
   │  Service │    │  Service │    │  Service │    │ Service  │
   └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
        │               │               │               │
        ▼               ▼               ▼               ▼
   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
   │  Users   │    │  Orders  │    │ Payments │    │ Inventory│
   │    DB    │    │    DB    │    │    DB    │    │    DB    │
   │ (Postgres│    │ (MongoDB)│    │ (Postgres│    │ (Redis + │
   │          │    │          │    │          │    │  MySQL)  │
   └──────────┘    └──────────┘    └──────────┘    └──────────┘

   Each service owns its OWN database.
   No service can directly access another service's database.

   ❓ But now how do you:
      • Do transactions across services?
      • JOIN data from multiple services?
      • Keep data consistent?
      • Handle partial failures?
```

---

## 🏛️ Pattern 1: Database Per Service

> **The foundational rule: Each microservice owns its database exclusively.**

### The Rules

```
┌──────────────────────────────────────────────────────────────────┐
│         DATABASE PER SERVICE — THE RULES                         │
│                                                                   │
│  Rule 1: Each service has its OWN database                       │
│          → User Service → users_db                                │
│          → Order Service → orders_db                              │
│          → They CAN be on the same DB server (different schemas) │
│                                                                   │
│  Rule 2: NO direct database access across services               │
│          → Order Service CANNOT query users_db directly           │
│          → Must go through User Service's API                     │
│                                                                   │
│  Rule 3: Each service can use a DIFFERENT database technology    │
│          → User Service: PostgreSQL (relational, strong schema)  │
│          → Product Catalog: MongoDB (flexible schema, fast reads) │
│          → Shopping Cart: Redis (in-memory, fast)                │
│          → Search: Elasticsearch (full-text search)              │
│          → This is called POLYGLOT PERSISTENCE                    │
│                                                                   │
│  Rule 4: Data WILL be duplicated across services                  │
│          → That's okay! Embrace denormalization.                  │
│          → Order Service stores user_name (not just user_id)     │
│          → This avoids runtime dependency on User Service         │
└──────────────────────────────────────────────────────────────────┘
```

### How Services Query Each Other's Data

```
  WRONG WAY ❌: Direct database access
  
  ┌───────────┐                    ┌──────────┐
  │  Order    │───── SQL query ───►│ users_db │  ← VIOLATION!
  │  Service  │                    └──────────┘
  └───────────┘

  RIGHT WAY ✅: API calls or Events

  Method 1: Synchronous API Call (REST/gRPC)
  ┌───────────┐    GET /users/42     ┌───────────┐    ┌──────────┐
  │  Order    │ ──────────────────► │   User    │───►│ users_db │
  │  Service  │ ◄────────────────── │  Service  │    └──────────┘
  └───────────┘   { name: "Alice" } └───────────┘

  ⚠️ Problem: If User Service is DOWN, Order Service can't work!

  Method 2: Event-Driven (Async messaging)
  ┌───────────┐                     ┌───────────┐
  │   User    │── UserCreated ────►│  Message   │
  │  Service  │   event            │  Broker    │
  └───────────┘                    │  (Kafka)   │
                                    └─────┬─────┘
                                          │
                                          ▼
                                   ┌───────────┐
                                   │  Order    │
                                   │  Service  │ ← Stores local copy
                                   └───────────┘   of user data

  ✅ Services are DECOUPLED
  ✅ Order Service has its OWN copy of user data
  ✅ Works even if User Service is temporarily down
```

### The Joins Problem — How to Query Across Services

```
  MONOLITH (easy):
  SELECT u.name, o.total, p.status
  FROM users u
  JOIN orders o ON u.id = o.user_id
  JOIN payments p ON o.id = p.order_id
  WHERE u.id = 42;

  MICROSERVICES (hard — three approaches):

  ═══════════════════════════════════════════════════════
  APPROACH 1: API Composition (Gateway Aggregation)
  ═══════════════════════════════════════════════════════

  ┌───────────────────┐
  │   API Gateway /   │
  │   Composition     │
  │   Service         │
  │                   │
  │  1. GET /users/42 │──► User Service ──► { name: "Alice" }
  │  2. GET /orders?  │──► Order Service ──► [{ total: 99 }]
  │     user=42       │
  │  3. GET /payments?│──► Payment Service ──► [{ status: "paid" }]
  │     order=123     │
  │                   │
  │  4. Combine in    │
  │     memory:       │
  │   {               │
  │     name: "Alice",│
  │     orders: [{    │
  │       total: 99,  │
  │       payment:    │
  │         "paid"    │
  │     }]            │
  │   }               │
  └───────────────────┘

  ✅ Simple to understand
  ❌ Multiple network calls (slow)
  ❌ Failures cascade
  ❌ Can't do WHERE across services efficiently

  ═══════════════════════════════════════════════════════
  APPROACH 2: CQRS — Materialized View
  ═══════════════════════════════════════════════════════

  Build a READ-OPTIMIZED view that combines data from all services.

  User Service ──► UserCreated event ──┐
                                        ├──► Query Service ──► orders_view
  Order Service ──► OrderPlaced event ──┤    (Pre-joined    │ { user_name,
                                        │     read model)   │   order_total,
  Payment Svc ──► PaymentCompleted ────┘                    │   payment_status }
                                                             └── Responds to
                                                                 complex queries!

  ✅ Fast reads (pre-computed)
  ✅ No runtime cross-service calls
  ❌ Eventually consistent (lag between event and view update)
  ❌ More infrastructure (event bus + query service)

  ═══════════════════════════════════════════════════════
  APPROACH 3: Data Replication (Denormalization)
  ═══════════════════════════════════════════════════════

  Each service stores copies of data it needs from other services.

  Order Service stores:
  {
      order_id: 123,
      user_id: 42,
      user_name: "Alice",      ← COPIED from User Service
      user_email: "a@b.com",   ← COPIED from User Service
      total: 99.99,
      payment_status: "paid"   ← COPIED from Payment Service
  }

  ✅ Fast (no cross-service calls for reads)
  ✅ Service works independently
  ❌ Data can be stale (user changes name, orders still show old name)
  ❌ Storage overhead
```

---

## 🎭 Pattern 2: The Saga Pattern

> **The solution to distributed transactions — replace ACID with a sequence of local transactions + compensating actions.**

### The Problem It Solves

```
  "Place an Order" requires 4 steps across 4 services:

  1. Order Service   → Create order
  2. Payment Service → Charge credit card
  3. Inventory Service → Reserve items
  4. Shipping Service → Schedule delivery

  In a monolith: One ACID transaction. Done.
  In microservices: Each step is a SEPARATE local transaction.

  What if Step 3 fails (out of stock)?
  → Step 1 already committed (order created)
  → Step 2 already committed (card charged!)
  → You MUST UNDO steps 1 and 2!

  A SAGA is a sequence of local transactions where each step
  has a COMPENSATING TRANSACTION that undoes its work.
```

### Saga Types

```
  ═══════════════════════════════════════════════════════
  TYPE 1: CHOREOGRAPHY (Event-Driven Saga)
  ═══════════════════════════════════════════════════════

  No central coordinator. Each service listens for events
  and reacts.

  HAPPY PATH:

  Order        Payment      Inventory     Shipping
  Service      Service      Service       Service
    │              │             │             │
    │─OrderCreated─►             │             │
    │              │             │             │
    │              │─PaymentCharged─►          │
    │              │             │             │
    │              │             │─ItemReserved─►
    │              │             │             │
    │              │             │     ShipScheduled
    │              │             │             │
    │◄──────── OrderConfirmed ─────────────────
    │              │             │             │

  FAILURE PATH (Inventory fails):

  Order        Payment      Inventory     Shipping
  Service      Service      Service       Service
    │              │             │             │
    │─OrderCreated─►             │             │
    │              │             │             │
    │              │─PaymentCharged─►          │
    │              │             │             │
    │              │    OutOfStock! ❌          │
    │              │             │             │
    │              │◄─RefundPayment            │
    │              │  (compensate)│             │
    │              │             │             │
    │◄─CancelOrder─             │             │
    │  (compensate)│             │             │

  ✅ Loosely coupled (no coordinator)
  ✅ Simple for few steps
  ❌ Hard to track overall saga state
  ❌ Cyclic dependencies possible
  ❌ Debugging is nightmare (which event triggered what?)

  ═══════════════════════════════════════════════════════
  TYPE 2: ORCHESTRATION (Central Coordinator)
  ═══════════════════════════════════════════════════════

  A Saga Orchestrator tells each service what to do.

  ┌──────────────────────────────────┐
  │        SAGA ORCHESTRATOR         │
  │     (Order Saga Coordinator)     │
  │                                  │
  │  State Machine:                  │
  │  ┌─────────┐                    │
  │  │ STARTED │                    │
  │  └────┬────┘                    │
  │       ▼                         │
  │  ┌──────────────┐              │
  │  │ PAYMENT_     │─── Fail ──►  COMPENSATING_PAYMENT │
  │  │ PENDING      │              │
  │  └────┬─────────┘              │
  │       ▼ Success                │
  │  ┌──────────────┐              │
  │  │ INVENTORY_   │─── Fail ──►  COMPENSATING_INVENTORY│
  │  │ PENDING      │              + REFUND_PAYMENT       │
  │  └────┬─────────┘              │
  │       ▼ Success                │
  │  ┌──────────────┐              │
  │  │ SHIPPING_    │─── Fail ──►  COMPENSATING_ALL      │
  │  │ PENDING      │              │
  │  └────┬─────────┘              │
  │       ▼ Success                │
  │  ┌──────────────┐              │
  │  │  COMPLETED   │              │
  │  └──────────────┘              │
  └──────────────────────────────────┘

  ✅ Clear control flow (state machine)
  ✅ Easy to understand and debug
  ✅ Handles complex business logic
  ❌ Orchestrator can become a single point of failure
  ❌ Tighter coupling (orchestrator knows all services)
```

### Saga Implementation Example

```python
# ═══════════════════════════════════════════════════════
# Orchestration Saga — Python Example
# ═══════════════════════════════════════════════════════

from enum import Enum
from dataclasses import dataclass
import httpx
import logging

class SagaState(Enum):
    STARTED = "STARTED"
    PAYMENT_PENDING = "PAYMENT_PENDING"
    PAYMENT_COMPLETED = "PAYMENT_COMPLETED"
    INVENTORY_PENDING = "INVENTORY_PENDING"
    INVENTORY_RESERVED = "INVENTORY_RESERVED"
    SHIPPING_PENDING = "SHIPPING_PENDING"
    COMPLETED = "COMPLETED"
    COMPENSATING = "COMPENSATING"
    FAILED = "FAILED"

@dataclass
class OrderSaga:
    order_id: str
    user_id: str
    items: list
    total: float
    state: SagaState = SagaState.STARTED

class OrderSagaOrchestrator:
    def __init__(self):
        self.payment_service = "http://payment-service:8080"
        self.inventory_service = "http://inventory-service:8080"
        self.shipping_service = "http://shipping-service:8080"
    
    async def execute(self, saga: OrderSaga):
        """Execute the saga — step by step with compensation on failure."""
        try:
            # Step 1: Charge payment
            saga.state = SagaState.PAYMENT_PENDING
            payment_id = await self._charge_payment(saga)
            saga.state = SagaState.PAYMENT_COMPLETED
            
            # Step 2: Reserve inventory
            saga.state = SagaState.INVENTORY_PENDING
            reservation_id = await self._reserve_inventory(saga)
            saga.state = SagaState.INVENTORY_RESERVED
            
            # Step 3: Schedule shipping
            saga.state = SagaState.SHIPPING_PENDING
            await self._schedule_shipping(saga)
            
            saga.state = SagaState.COMPLETED
            logging.info(f"Saga {saga.order_id} completed successfully!")
            
        except PaymentError:
            # Payment failed — nothing to compensate
            saga.state = SagaState.FAILED
            
        except InventoryError:
            # Inventory failed — compensate payment
            saga.state = SagaState.COMPENSATING
            await self._refund_payment(payment_id)
            saga.state = SagaState.FAILED
            
        except ShippingError:
            # Shipping failed — compensate inventory + payment
            saga.state = SagaState.COMPENSATING
            await self._release_inventory(reservation_id)
            await self._refund_payment(payment_id)
            saga.state = SagaState.FAILED
    
    async def _charge_payment(self, saga):
        async with httpx.AsyncClient() as client:
            resp = await client.post(f"{self.payment_service}/charge", 
                json={"user_id": saga.user_id, "amount": saga.total})
            if resp.status_code != 200:
                raise PaymentError(resp.text)
            return resp.json()["payment_id"]
    
    async def _refund_payment(self, payment_id):
        """Compensating transaction for payment"""
        async with httpx.AsyncClient() as client:
            await client.post(f"{self.payment_service}/refund",
                json={"payment_id": payment_id})
    
    async def _reserve_inventory(self, saga):
        async with httpx.AsyncClient() as client:
            resp = await client.post(f"{self.inventory_service}/reserve",
                json={"items": saga.items})
            if resp.status_code != 200:
                raise InventoryError(resp.text)
            return resp.json()["reservation_id"]
    
    async def _release_inventory(self, reservation_id):
        """Compensating transaction for inventory"""
        async with httpx.AsyncClient() as client:
            await client.post(f"{self.inventory_service}/release",
                json={"reservation_id": reservation_id})
    
    async def _schedule_shipping(self, saga):
        async with httpx.AsyncClient() as client:
            resp = await client.post(f"{self.shipping_service}/schedule",
                json={"order_id": saga.order_id, "items": saga.items})
            if resp.status_code != 200:
                raise ShippingError(resp.text)
```

### Saga Isolation — The Dirty Secret

```
┌──────────────────────────────────────────────────────────────────┐
│  ⚠️ SAGAS DON'T PROVIDE ISOLATION!                              │
│                                                                   │
│  While a saga is running, other sagas can see intermediate state:│
│                                                                   │
│  Saga 1: Create Order → Charge Payment → Reserve Inventory      │
│                                    ↑                             │
│                              HERE: Payment charged but           │
│                              inventory NOT yet reserved           │
│                                                                   │
│  Saga 2 runs at this moment:                                     │
│  → Sees the payment exists                                       │
│  → But inventory isn't reserved yet                               │
│  → Reads inconsistent state! 😱                                  │
│                                                                   │
│  COUNTERMEASURES:                                                 │
│                                                                   │
│  1. SEMANTIC LOCK (Status field)                                  │
│     → Order has status: "PENDING" during saga                    │
│     → Other sagas skip orders in "PENDING" state                 │
│     → Set to "CONFIRMED" only after saga completes              │
│                                                                   │
│  2. COMMUTATIVE UPDATES                                           │
│     → Design updates so order doesn't matter                     │
│     → Increment/decrement instead of absolute sets               │
│                                                                   │
│  3. PESSIMISTIC VIEW                                              │
│     → Reorder saga steps: do risky steps first                   │
│     → If inventory is limited, check it BEFORE charging          │
│                                                                   │
│  4. REREAD VALUE                                                  │
│     → Before compensating, re-read the value                     │
│     → Verify it hasn't changed since you wrote it                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📸 Pattern 3: Event Sourcing

> **Instead of storing current STATE, store every EVENT that ever happened. Reconstruct state by replaying events.**

### Traditional vs Event Sourcing

```
  ═══════════════════════════════════════════════════════
  TRADITIONAL (State-Based)
  ═══════════════════════════════════════════════════════

  Bank Account Table:
  ┌────────────┬──────────┬─────────┐
  │ account_id │ balance  │ updated │
  ├────────────┼──────────┼─────────┤
  │ ACC-001    │ $1,500   │ today   │  ← Only CURRENT state
  └────────────┴──────────┴─────────┘

  What happened to get to $1,500? 🤷 No idea.

  ═══════════════════════════════════════════════════════
  EVENT SOURCING
  ═══════════════════════════════════════════════════════

  Event Store:
  ┌────┬─────────────┬────────────────┬─────────┬──────────┐
  │ #  │ account_id  │ event_type     │ amount  │ timestamp│
  ├────┼─────────────┼────────────────┼─────────┼──────────┤
  │ 1  │ ACC-001     │ AccountCreated │ $0      │ Jan 1    │
  │ 2  │ ACC-001     │ MoneyDeposited │ +$2,000 │ Jan 5    │
  │ 3  │ ACC-001     │ MoneyWithdrawn │ -$300   │ Jan 10   │
  │ 4  │ ACC-001     │ MoneyWithdrawn │ -$200   │ Jan 15   │
  │ 5  │ ACC-001     │ MoneyDeposited │ +$500   │ Jan 20   │
  │ 6  │ ACC-001     │ MoneyWithdrawn │ -$500   │ Jan 25   │
  └────┴─────────────┴────────────────┴─────────┴──────────┘

  Current balance = Replay all events:
  $0 + $2,000 - $300 - $200 + $500 - $500 = $1,500 ✅

  FULL AUDIT TRAIL! 📜
  Can reconstruct state at ANY point in time!
  "What was the balance on Jan 12?" → Replay events 1-3 = $1,700
```

### Why Event Sourcing?

```
┌──────────────────────────────────────────────────────────────────┐
│  ✅ BENEFITS:                                                     │
│                                                                   │
│  1. COMPLETE AUDIT TRAIL                                          │
│     → Every change ever made is recorded                          │
│     → Perfect for finance, healthcare, legal                      │
│                                                                   │
│  2. TIME TRAVEL                                                   │
│     → Reconstruct state at any past moment                        │
│     → "What did the inventory look like on Black Friday?"         │
│                                                                   │
│  3. DEBUGGING SUPERPOWER                                          │
│     → Replay events to reproduce bugs                             │
│     → "What exactly happened before the crash?"                   │
│                                                                   │
│  4. EVENT REPLAY                                                  │
│     → New service? Replay all events to build its state          │
│     → Fix a bug in event handler? Replay and recompute!          │
│                                                                   │
│  5. NATURAL FIT FOR MICROSERVICES                                 │
│     → Events ARE the integration between services                │
│     → Publish domain events → other services react               │
│                                                                   │
│  ❌ CHALLENGES:                                                    │
│                                                                   │
│  1. QUERY COMPLEXITY                                              │
│     → "What's the current balance?" requires replaying events    │
│     → Solution: Materialized views / CQRS (read projections)    │
│                                                                   │
│  2. EVENTUAL CONSISTENCY                                          │
│     → Event → Projection update has a lag                        │
│                                                                   │
│  3. EVENT SCHEMA EVOLUTION                                        │
│     → What if event format changes? (upcasting/versioning)       │
│                                                                   │
│  4. STORAGE                                                       │
│     → Events accumulate forever (snapshots help)                 │
│                                                                   │
│  5. STEEP LEARNING CURVE                                          │
│     → Fundamentally different way of thinking about data         │
└──────────────────────────────────────────────────────────────────┘
```

### Event Sourcing Implementation

```python
# ═══════════════════════════════════════════════════════
# Event Sourcing — Python Example
# ═══════════════════════════════════════════════════════

from dataclasses import dataclass, field
from datetime import datetime
from typing import List
import json

# ─── Events (Immutable Facts) ─────────────────────────

@dataclass(frozen=True)
class AccountCreated:
    account_id: str
    owner: str
    timestamp: datetime

@dataclass(frozen=True)
class MoneyDeposited:
    account_id: str
    amount: float
    timestamp: datetime

@dataclass(frozen=True)
class MoneyWithdrawn:
    account_id: str
    amount: float
    timestamp: datetime

# ─── Aggregate (Rebuilds state from events) ────────────

class BankAccount:
    def __init__(self, account_id: str):
        self.account_id = account_id
        self.balance = 0.0
        self.owner = ""
        self.is_open = False
        self._events: List = []
    
    # ── Commands (things the user wants to do) ──
    
    def create(self, owner: str):
        if self.is_open:
            raise ValueError("Account already exists")
        self._apply(AccountCreated(self.account_id, owner, datetime.now()))
    
    def deposit(self, amount: float):
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self._apply(MoneyDeposited(self.account_id, amount, datetime.now()))
    
    def withdraw(self, amount: float):
        if amount > self.balance:
            raise ValueError("Insufficient funds")
        self._apply(MoneyWithdrawn(self.account_id, amount, datetime.now()))
    
    # ── Event Handlers (how events change state) ──
    
    def _on_account_created(self, event: AccountCreated):
        self.owner = event.owner
        self.is_open = True
        self.balance = 0.0
    
    def _on_money_deposited(self, event: MoneyDeposited):
        self.balance += event.amount
    
    def _on_money_withdrawn(self, event: MoneyWithdrawn):
        self.balance -= event.amount
    
    # ── Infrastructure ──
    
    def _apply(self, event):
        """Apply event to state and record it."""
        handler = {
            AccountCreated: self._on_account_created,
            MoneyDeposited: self._on_money_deposited,
            MoneyWithdrawn: self._on_money_withdrawn,
        }[type(event)]
        handler(event)
        self._events.append(event)
    
    @classmethod
    def from_events(cls, account_id: str, events: List):
        """Reconstruct account from event history."""
        account = cls(account_id)
        for event in events:
            account._apply(event)
        account._events.clear()  # Don't re-record old events
        return account


# ─── Usage ─────────────────────────────────────────────

# Create and use account
account = BankAccount("ACC-001")
account.create("Alice")
account.deposit(2000)
account.withdraw(300)
account.withdraw(200)
account.deposit(500)

print(f"Balance: ${account.balance}")  # $2,000
print(f"Events:  {len(account._events)}")  # 5 events

# Later: Reconstruct from stored events
stored_events = account._events
rebuilt = BankAccount.from_events("ACC-001", stored_events)
print(f"Rebuilt balance: ${rebuilt.balance}")  # $2,000 ✅ Same!
```

### Snapshots — Because Replaying 10 Million Events Is Slow

```
  Problem: Account has 10 million events → replaying takes forever!

  Solution: SNAPSHOTS

  Events: [E1, E2, E3, ... E5000000, E5000001, ... E10000000]
                              ↑
                        SNAPSHOT HERE!
                     { balance: $45,231.50,
                       snapshot_at_event: 5000000 }

  To rebuild current state:
  1. Load snapshot (balance = $45,231.50 at event 5M)
  2. Replay only events 5,000,001 → 10,000,000
  3. That's 5M events instead of 10M! ⚡

  ┌─────────────────────────────────────────────────────────┐
  │  SNAPSHOT STRATEGIES:                                    │
  │                                                          │
  │  1. Every N events (e.g., every 1,000 events)           │
  │  2. Every T minutes (e.g., every hour)                  │
  │  3. On demand (when replay time exceeds threshold)      │
  │  4. Keep last 3 snapshots (for rollback safety)         │
  └─────────────────────────────────────────────────────────┘
```

---

## 🔀 Pattern 4: CQRS (Command Query Responsibility Segregation)

> **Separate the WRITE model from the READ model. Different databases, different schemas, different optimizations.**

### The Core Idea

```
                    TRADITIONAL APPROACH
                    ═══════════════════

                  ┌────────────────────────┐
                  │    Application         │
                  │                        │
                  │  Same Model for:       │
                  │  • INSERT/UPDATE/DELETE │
                  │  • SELECT queries       │
                  │                        │
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │    ONE DATABASE         │
                  │    ONE SCHEMA           │
                  │    Optimized for ???    │
                  └────────────────────────┘

                    CQRS APPROACH
                    ═════════════

            ┌──────────────┐         ┌──────────────┐
            │   COMMAND     │         │    QUERY     │
            │   (Write)     │         │    (Read)    │
            │               │         │              │
            │  CreateOrder  │         │  GetOrders   │
            │  UpdateOrder  │         │  SearchOrder │
            │  CancelOrder  │         │  Dashboard   │
            └──────┬───────┘         └──────┬───────┘
                   │                        │
                   ▼                        ▼
            ┌──────────────┐         ┌──────────────┐
            │  WRITE DB    │────────►│  READ DB     │
            │  (PostgreSQL)│  events │ (Elasticsearch│
            │              │  / CDC  │  or MongoDB   │
            │  Normalized  │         │  or Redis)    │
            │  ACID        │         │               │
            │  Write-      │         │  Denormalized │
            │  optimized   │         │  Pre-computed │
            │              │         │  Read-        │
            │              │         │  optimized    │
            └──────────────┘         └──────────────┘

  ✅ Write DB: Normalized, strong consistency, ACID
  ✅ Read DB: Denormalized, fast queries, pre-joined data
  ✅ Each optimized for its specific purpose!
```

### When CQRS Makes Sense

```
┌──────────────────────────────────────────────────────────────────┐
│  ✅ USE CQRS WHEN:                                               │
│                                                                   │
│  • Read and write patterns are VERY different                    │
│    (e.g., write normalized, read denormalized)                   │
│  • Read/write ratio is highly skewed (1000:1 reads to writes)   │
│  • Different scaling needs (scale reads independently)           │
│  • Complex queries that don't fit the write model                │
│  • Combined with Event Sourcing                                   │
│                                                                   │
│  ❌ DON'T USE CQRS WHEN:                                         │
│                                                                   │
│  • Simple CRUD app (overkill)                                    │
│  • Read and write models are similar                              │
│  • Team is small / inexperienced with distributed systems        │
│  • Strong consistency is required everywhere                      │
│  • You can't tolerate eventual consistency on reads               │
└──────────────────────────────────────────────────────────────────┘
```

### CQRS + Event Sourcing — The Power Combo

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │    Command                Event                  Query       │
  │    ─────────              Store                  ─────       │
  │                                                              │
  │    PlaceOrder ──────► ┌──────────┐                          │
  │                       │ OrderPlaced               │          │
  │    CancelOrder ─────► │ OrderCancelled            │          │
  │                       │ ItemShipped               │          │
  │    ShipItem ────────► │ PaymentReceived           │          │
  │                       │ ...                       │          │
  │                       └──────┬───────────┘        │          │
  │                              │                    │          │
  │                    Events published to:            │          │
  │                              │                    │          │
  │              ┌───────────────┼────────────────┐   │          │
  │              │               │                │   │          │
  │              ▼               ▼                ▼   │          │
  │       ┌──────────┐   ┌──────────┐    ┌──────────┐│          │
  │       │ Orders   │   │ Customer │    │ Analytics││          │
  │       │ Read     │   │ History  │    │ Read     ││          │
  │       │ Model    │   │ Read     │    │ Model    ││          │
  │       │(Postgres)│   │ Model    │    │(ClickHse)││          │
  │       │          │   │ (MongoDB)│    │          ││          │
  │       └──────────┘   └──────────┘    └──────────┘│          │
  │            │              │               │       │          │
  │            │      ◄────── QUERIES ───────►│       │          │
  │            │                              │       │          │
  │       "Show my         "Full order       "Revenue│          │
  │        orders"          history"          report" │          │
  │                                                   │          │
  └─────────────────────────────────────────────────────────────┘

  Each read model is a PROJECTION — a materialized view
  optimized for a specific query pattern.
  
  New reporting requirement? Create a NEW read model
  by replaying all events! No schema migration needed!
```

---

## 📤 Pattern 5: The Transactional Outbox Pattern

> **The solution to the "dual write" problem — reliably publish events AND update the database atomically.**

### The Dual Write Problem

```
  PROBLEM: Service needs to do TWO things:
  
  1. Write to database (create order)
  2. Publish event to message broker (OrderCreated)

  ┌──────────────────────────────────────────────────────────────┐
  │  WHAT CAN GO WRONG:                                          │
  │                                                               │
  │  Scenario A: DB succeeds, Message fails                       │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
  │  │  Service  │──►│ Database │ ✅ │          │               │
  │  │          │──►│          │    │  Kafka   │ ❌ (down!)     │
  │  └──────────┘    └──────────┘    └──────────┘               │
  │  → Order created but other services never notified!          │
  │                                                               │
  │  Scenario B: Message succeeds, DB fails                       │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
  │  │  Service  │──►│ Database │ ❌ (crashed!)│               │
  │  │          │──►│          │    │  Kafka   │ ✅             │
  │  └──────────┘    └──────────┘    └──────────┘               │
  │  → Event published but order doesn't exist in DB!            │
  │                                                               │
  │  You CAN'T do an ACID transaction across DB + Kafka!         │
  │  They're different systems!                                   │
  └──────────────────────────────────────────────────────────────┘
```

### The Outbox Solution

```
  SOLUTION: Write the event INTO the database,
            in the SAME transaction as the business data.
            A separate process reads the outbox and publishes.

  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │   ┌──────────┐                                                │
  │   │  Service  │                                                │
  │   └────┬─────┘                                                │
  │        │                                                      │
  │        │ BEGIN TRANSACTION;                                    │
  │        │                                                      │
  │        ▼                                                      │
  │   ┌──────────────────────────────────┐                       │
  │   │          DATABASE                │                       │
  │   │                                  │                       │
  │   │  1. INSERT INTO orders (...);    │                       │
  │   │                                  │                       │
  │   │  2. INSERT INTO outbox (         │ ← Same transaction! │
  │   │       event_type, payload,       │                       │
  │   │       created_at                 │                       │
  │   │     );                           │                       │
  │   │                                  │                       │
  │   │  COMMIT; ← ATOMIC! Both succeed │                       │
  │   │          or both fail            │                       │
  │   └──────────────┬───────────────────┘                       │
  │                  │                                            │
  │   ┌──────────────▼───────────────────┐                       │
  │   │    OUTBOX PUBLISHER              │                       │
  │   │    (Separate process)             │                       │
  │   │                                  │                       │
  │   │  1. Poll outbox table for new   │                       │
  │   │     events                       │                       │
  │   │  2. Publish to Kafka             │                       │
  │   │  3. Mark events as published     │                       │
  │   │                                  │                       │
  │   │  OR use CDC (Debezium) to       │                       │
  │   │  capture outbox changes          │                       │
  │   │  automatically! ✅               │                       │
  │   └──────────────┬───────────────────┘                       │
  │                  │                                            │
  │                  ▼                                            │
  │   ┌──────────────────────────────────┐                       │
  │   │          KAFKA / EVENT BUS       │                       │
  │   │                                  │                       │
  │   │  OrderCreated event published    │                       │
  │   │  reliably! ✅                    │                       │
  │   └──────────────────────────────────┘                       │
  │                                                               │
  └──────────────────────────────────────────────────────────────┘
```

### Outbox Table Schema

```sql
-- ═══════════════════════════════════════════════════════
-- Outbox Table Design
-- ═══════════════════════════════════════════════════════

CREATE TABLE outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(255) NOT NULL,  -- e.g., 'Order'
    aggregate_id    VARCHAR(255) NOT NULL,  -- e.g., 'ORD-42'
    event_type      VARCHAR(255) NOT NULL,  -- e.g., 'OrderCreated'
    payload         JSONB NOT NULL,         -- Event data
    created_at      TIMESTAMP DEFAULT NOW(),
    published_at    TIMESTAMP NULL          -- NULL = not yet published
);

-- Create index for polling unpublished events
CREATE INDEX idx_outbox_unpublished 
    ON outbox (created_at) 
    WHERE published_at IS NULL;

-- ═══════════════════════════════════════════════════════
-- Business Operation + Outbox (Same Transaction!)
-- ═══════════════════════════════════════════════════════

BEGIN;

-- Step 1: Business operation
INSERT INTO orders (order_id, user_id, total, status)
VALUES ('ORD-42', 'USER-7', 299.99, 'CREATED');

-- Step 2: Write event to outbox (same transaction!)
INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
VALUES (
    'Order',
    'ORD-42',
    'OrderCreated',
    '{
        "order_id": "ORD-42",
        "user_id": "USER-7",
        "total": 299.99,
        "items": [{"product_id": "P1", "qty": 2}],
        "timestamp": "2025-06-01T10:30:00Z"
    }'::jsonb
);

COMMIT;  -- Both succeed or both fail! ✅

-- ═══════════════════════════════════════════════════════
-- Outbox Publisher (Polling approach)
-- ═══════════════════════════════════════════════════════

-- 1. Fetch unpublished events (with lock to prevent duplicates)
SELECT id, event_type, payload 
FROM outbox 
WHERE published_at IS NULL 
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;

-- 2. After successfully publishing to Kafka:
UPDATE outbox SET published_at = NOW() WHERE id = 'event-uuid';

-- 3. Periodically clean up old published events:
DELETE FROM outbox WHERE published_at < NOW() - INTERVAL '7 days';
```

### CDC-Based Outbox (The Better Way)

```
  Instead of polling the outbox table, use CDC (Debezium)
  to capture changes automatically!

  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │ Service  │────────►│ Database │         │          │
  │          │ writes  │          │         │  Kafka   │
  │          │ to DB + │ outbox   │         │          │
  │          │ outbox  │ table    │         │          │
  └──────────┘         └────┬─────┘         └────▲─────┘
                            │                    │
                            │                    │
                       ┌────▼─────────────────────┐
                       │      Debezium CDC        │
                       │                          │
                       │  Reads WAL / Binlog      │
                       │  Captures outbox inserts │
                       │  Publishes to Kafka      │
                       │  Exactly-once semantics! │
                       └──────────────────────────┘

  ✅ No polling (real-time via WAL/binlog)
  ✅ Lower latency than polling
  ✅ Debezium handles deduplication
  ✅ Battle-tested at scale
```

---

## 🗺️ Pattern Comparison — When to Use What

```
┌───────────────────┬──────────────────┬─────────────────┬─────────────┐
│     Pattern       │   Use When       │   Complexity    │  Consistency│
├───────────────────┼──────────────────┼─────────────────┼─────────────┤
│ DB per Service    │ Always (default  │ 🟡 Medium       │ Eventual    │
│                   │ in microservices)│                 │             │
│                   │                  │                 │             │
│ Saga              │ Cross-service    │ 🔴 High         │ Eventual    │
│ (Choreography)    │ transactions,    │ (debugging!)    │             │
│                   │ few steps        │                 │             │
│                   │                  │                 │             │
│ Saga              │ Cross-service    │ 🟡 Medium       │ Eventual    │
│ (Orchestration)   │ transactions,    │ (clear flow)    │             │
│                   │ complex logic    │                 │             │
│                   │                  │                 │             │
│ Event Sourcing    │ Audit trail,     │ 🔴 High         │ Eventual    │
│                   │ time travel,     │ (paradigm shift)│             │
│                   │ finance/legal    │                 │             │
│                   │                  │                 │             │
│ CQRS              │ Different read/  │ 🟡 Medium       │ Eventual    │
│                   │ write needs,     │ (more infra)    │ (reads)     │
│                   │ read-heavy apps  │                 │             │
│                   │                  │                 │             │
│ Outbox Pattern    │ Reliable event   │ 🟢 Low          │ At-least-   │
│                   │ publishing,      │ (just a table!) │ once        │
│                   │ always           │                 │             │
│                   │                  │                 │             │
│ API Composition   │ Simple cross-    │ 🟢 Low          │ Depends on  │
│                   │ service queries  │                 │ services    │
└───────────────────┴──────────────────┴─────────────────┴─────────────┘
```

---

## 🏢 Real-World Architecture Examples

### Uber — Polyglot Persistence

```
┌──────────────────────────────────────────────────────────────┐
│  UBER'S DATABASE ARCHITECTURE:                                │
│                                                               │
│  Trip Service      → MySQL (sharded by city)                 │
│  Matching Service  → Redis (in-memory, fast matching)        │
│  Maps/ETA Service  → PostgreSQL + PostGIS (geospatial)       │
│  Payment Service   → MySQL (ACID for money)                  │
│  Surge Pricing     → Cassandra (high write throughput)       │
│  Analytics         → Apache Hive / Presto (data lake)        │
│  Search            → Elasticsearch (location search)         │
│  Message Queue     → Apache Kafka (event streaming)          │
│                                                               │
│  Pattern: CQRS + Event Sourcing for trip lifecycle           │
│  Pattern: Saga for ride booking (match → pay → rate)         │
│  Pattern: Outbox for reliable payment notifications          │
└──────────────────────────────────────────────────────────────┘
```

### Netflix — Event-Driven Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  NETFLIX'S DATABASE ARCHITECTURE:                             │
│                                                               │
│  User Profiles     → Cassandra (globally distributed)        │
│  Viewing History   → Cassandra (massive write volume)        │
│  Recommendations   → Cassandra + custom ML pipeline          │
│  Content Metadata  → MySQL (migrating to CockroachDB)       │
│  Billing/Payments  → MySQL (ACID, financial data)            │
│  Search            → Elasticsearch                            │
│  Caching           → EVCache (Memcached-based)               │
│  Data Warehouse    → Apache Spark + S3 (analytics)           │
│  Event Bus         → Apache Kafka (100B+ events/day)         │
│                                                               │
│  Key Insight:                                                 │
│  → Cassandra for AP (availability + partition tolerance)     │
│  → MySQL for CP (consistency + partition tolerance)          │
│  → Pick the right DB for the right use case!                 │
└──────────────────────────────────────────────────────────────┘
```

---

## ⚠️ Common Microservices Database Anti-Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│  🚫 ANTI-PATTERN 1: Shared Database                              │
│     → Multiple services share ONE database                       │
│     → Changes to schema break OTHER services                     │
│     → Can't deploy services independently                        │
│     → FIX: Database per service                                   │
│                                                                   │
│  🚫 ANTI-PATTERN 2: Distributed Monolith                         │
│     → Services call each other synchronously in chains           │
│     → A → B → C → D → if D is slow, A is slow                   │
│     → FIX: Async communication + event-driven                    │
│                                                                   │
│  🚫 ANTI-PATTERN 3: No Data Ownership                            │
│     → Multiple services can modify the same data                 │
│     → No single source of truth                                   │
│     → FIX: Each piece of data has ONE owning service             │
│                                                                   │
│  🚫 ANTI-PATTERN 4: Distributed Transactions (2PC everywhere)   │
│     → Using Two-Phase Commit across services                     │
│     → Extremely slow, fragile, doesn't scale                    │
│     → FIX: Sagas + eventual consistency                           │
│                                                                   │
│  🚫 ANTI-PATTERN 5: Too Fine-Grained Services                   │
│     → Splitting too small → massive coordination overhead        │
│     → "User Address Service"? Just keep it in User Service!      │
│     → FIX: Align services with business domains (DDD)            │
│                                                                   │
│  🚫 ANTI-PATTERN 6: Ignoring Data Consistency                    │
│     → "It's eventual consistency, it'll work itself out"         │
│     → Without proper patterns, data WILL get corrupted           │
│     → FIX: Explicit consistency patterns (Saga, Outbox, etc.)   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Decision Framework — Choosing Patterns

```
                    ┌──────────────────────┐
                    │  Do you need cross-  │
                    │  service consistency?│
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │         NO           │──► Keep it simple.
                    │                      │    Database per service.
                    └──────────────────────┘    Local transactions.
                    ┌──────────▼───────────┐
                    │         YES          │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Can you tolerate    │
                    │  eventual consistency?│
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │ YES            │                │ NO
              ▼                │                ▼
     ┌────────────────┐        │       ┌─────────────────┐
     │  Saga Pattern  │        │       │ Consider if you  │
     │                │        │       │ truly need       │
     │ Few steps?     │        │       │ microservices    │
     │ → Choreography │        │       │ for this feature │
     │                │        │       │                  │
     │ Complex logic? │        │       │ OR use NewSQL    │
     │ → Orchestration│        │       │ (CockroachDB,    │
     └────────────────┘        │       │  Spanner)        │
              │                │       └─────────────────┘
              │                │
              ▼                │
     ┌────────────────┐        │
     │ Need reliable  │        │
     │ event publish? │        │
     └──────┬─────────┘        │
            │ YES              │
            ▼                  │
     ┌────────────────┐        │
     │ Outbox Pattern │        │
     │ + CDC/Debezium │        │
     └────────────────┘        │
                               │
              Need audit trail / time travel?
                    │ YES
                    ▼
            ┌────────────────┐
            │ Event Sourcing │
            │ + CQRS         │
            └────────────────┘
```

---

## 🔗 What's Next?

Now that you understand how databases work in distributed architectures, the next chapter covers **Caching Strategies** — the performance multiplier that sits between your application and database.

> **Chapter 7.4:** [Caching Strategies with Databases →](./04-Caching-Strategies.md)

---

> _"A distributed system is one where the failure of a computer you've never heard of can prevent you from doing your work."_
> — Leslie Lamport (Turing Award winner, inventor of Paxos)
