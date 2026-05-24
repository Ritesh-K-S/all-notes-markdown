# Saga Pattern — Managing Distributed Transactions

> **What you'll learn**: How to maintain data consistency across multiple microservices when traditional database transactions (ACID) are impossible — using choreography and orchestration to coordinate multi-step business processes.

---

## Real-Life Analogy

Imagine booking a vacation package that involves:
1. **Booking a flight** (airline)
2. **Reserving a hotel** (hotel chain)
3. **Renting a car** (car rental company)

These are three separate companies. There's no single "transaction" that can atomically book all three. So what happens if the car rental fails after the flight and hotel are already booked?

You need **compensating actions** — cancel the flight, cancel the hotel.

That's exactly what a Saga does in software. It's a sequence of local transactions where each step either succeeds and triggers the next step, or fails and triggers compensating (undo) actions for all previous steps.

```
HAPPY PATH:
Book Flight ──▶ Reserve Hotel ──▶ Rent Car ──▶ ✅ All Done!

FAILURE + COMPENSATION:
Book Flight ──▶ Reserve Hotel ──▶ Rent Car ──▶ ❌ Failed!
                                       │
                                       ▼
                              Cancel Hotel ──▶ Cancel Flight
                              (Compensate)     (Compensate)
```

---

## Why Do We Need Sagas?

In a monolith with a single database, we can use **ACID transactions**:

```
BEGIN TRANSACTION;
  INSERT INTO orders (...);
  UPDATE inventory SET stock = stock - 1;
  INSERT INTO payments (...);
COMMIT;  ← All or nothing!
```

But in microservices, each service has its **own database**:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Order Service │    │  Inventory   │    │   Payment    │
│              │    │   Service    │    │   Service    │
├──────────────┤    ├──────────────┤    ├──────────────┤
│  Orders DB   │    │ Inventory DB │    │ Payments DB  │
└──────────────┘    └──────────────┘    └──────────────┘

❌ Cannot do a single transaction across 3 databases!
❌ 2PC (Two-Phase Commit) doesn't scale
✅ Saga = sequence of local transactions + compensations
```

---

## Two Types of Sagas

### Type 1: Choreography-Based Saga (Event-Driven)

Each service listens for events and decides what to do next. No central coordinator.

```
┌─────────────────────────────────────────────────────────────────┐
│              CHOREOGRAPHY SAGA - Order Flow                       │
│                                                                   │
│  ┌─────────┐   OrderCreated   ┌───────────┐  StockReserved      │
│  │  Order  │──────────────────▶│ Inventory │─────────────────┐   │
│  │ Service │                   │  Service  │                 │   │
│  └─────────┘                   └───────────┘                 │   │
│       ▲                                                      ▼   │
│       │                                              ┌───────────┐│
│       │            PaymentProcessed                   │  Payment  ││
│       └──────────────────────────────────────────────│  Service  ││
│                                                      └───────────┘│
│                                                                   │
│  Each service:                                                    │
│  1. Does its local transaction                                    │
│  2. Publishes an event                                            │
│  3. Next service reacts to the event                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Happy Path:**
```
Order Service           Inventory Service         Payment Service
     │                        │                        │
     │──OrderCreated─────────▶│                        │
     │                        │                        │
     │                        │──StockReserved────────▶│
     │                        │                        │
     │◀──PaymentProcessed─────┼────────────────────────│
     │                        │                        │
     │ (Mark order CONFIRMED) │                        │
```

**Failure + Compensation:**
```
Order Service           Inventory Service         Payment Service
     │                        │                        │
     │──OrderCreated─────────▶│                        │
     │                        │                        │
     │                        │──StockReserved────────▶│
     │                        │                        │
     │                        │◀──PaymentFailed────────│
     │                        │                        │
     │◀──StockReleased────────│  (Compensate:          │
     │                        │   release reserved     │
     │ (Mark order FAILED)    │   stock)               │
```

---

### Type 2: Orchestration-Based Saga (Central Coordinator)

A **Saga Orchestrator** tells each service what to do and handles failures centrally.

```
┌─────────────────────────────────────────────────────────────────┐
│              ORCHESTRATION SAGA - Order Flow                      │
│                                                                   │
│                    ┌────────────────────┐                         │
│                    │  SAGA ORCHESTRATOR │                         │
│                    │  (Order Saga)      │                         │
│                    └─────────┬──────────┘                         │
│                              │                                    │
│              ┌───────────────┼───────────────┐                    │
│              │               │               │                    │
│              ▼               ▼               ▼                    │
│        ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│        │ Inventory│   │ Payment  │   │ Shipping │               │
│        │ Service  │   │ Service  │   │ Service  │               │
│        └──────────┘   └──────────┘   └──────────┘               │
│                                                                   │
│  Orchestrator:                                                    │
│  1. "Inventory, reserve stock" → Success                         │
│  2. "Payment, charge customer" → Success                         │
│  3. "Shipping, create shipment" → Success                        │
│  4. Mark saga COMPLETED                                           │
│                                                                   │
│  If step 2 fails:                                                │
│  1. "Inventory, RELEASE stock" (compensate)                      │
│  2. Mark saga FAILED                                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Choreography vs Orchestration — When to Use Which?

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Coordination | Decentralized (events) | Centralized (orchestrator) |
| Coupling | Loose (services don't know each other) | Tighter (orchestrator knows all services) |
| Complexity | Hard to track flow across services | Easy to see full flow in one place |
| Failure handling | Each service handles its own compensation | Orchestrator manages all compensations |
| Best for | Simple flows (2-3 steps) | Complex flows (4+ steps, branching) |
| Debugging | Hard (distributed event trail) | Easy (check orchestrator state) |
| Single point of failure | None | Orchestrator |

---

## How It Works Internally — Saga State Machine

A saga is essentially a **state machine** that transitions between states:

```
┌─────────────────────────────────────────────────────────────┐
│              ORDER SAGA STATE MACHINE                         │
│                                                               │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ STARTED  │───▶│RESERVING_STOCK│───▶│  CHARGING    │       │
│  └──────────┘    └──────┬───────┘    └──────┬───────┘       │
│                         │                    │                │
│                    ┌────┘               ┌────┘                │
│                    ▼ fail               ▼ fail                │
│              ┌──────────┐        ┌──────────────┐            │
│              │  FAILED  │◀───────│COMPENSATING  │            │
│              └──────────┘        └──────────────┘            │
│                                        │ success             │
│                    ┌───────────────────┘                     │
│                    ▼                                          │
│              ┌──────────┐                                    │
│              │COMPLETED │ (Happy path from CHARGING)          │
│              └──────────┘                                    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

The orchestrator persists the saga state in a database, so if the orchestrator crashes and restarts, it can resume from where it left off.

---

## Code Examples

### Python: Orchestration-Based Saga

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Callable
import uuid

class SagaStatus(Enum):
    STARTED = "STARTED"
    EXECUTING = "EXECUTING"
    COMPENSATING = "COMPENSATING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"

@dataclass
class SagaStep:
    name: str
    action: Callable          # Forward action
    compensation: Callable    # Undo action
    completed: bool = False

@dataclass
class OrderSaga:
    saga_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    order_id: str = ""
    status: SagaStatus = SagaStatus.STARTED
    current_step: int = 0
    steps: List[SagaStep] = field(default_factory=list)
    
    def execute(self):
        """Execute saga steps sequentially"""
        self.status = SagaStatus.EXECUTING
        
        for i, step in enumerate(self.steps):
            self.current_step = i
            try:
                print(f"  Executing: {step.name}")
                step.action()
                step.completed = True
            except Exception as e:
                print(f"  ❌ Failed at: {step.name} - {e}")
                self._compensate()
                return False
        
        self.status = SagaStatus.COMPLETED
        print(f"  ✅ Saga completed successfully!")
        return True
    
    def _compensate(self):
        """Undo all completed steps in reverse order"""
        self.status = SagaStatus.COMPENSATING
        print(f"  🔄 Starting compensation...")
        
        # Compensate completed steps in reverse order
        for i in range(self.current_step - 1, -1, -1):
            step = self.steps[i]
            if step.completed:
                try:
                    print(f"  Compensating: {step.name}")
                    step.compensation()
                except Exception as e:
                    # Log and continue - compensation should be retried
                    print(f"  ⚠️ Compensation failed for {step.name}: {e}")
        
        self.status = SagaStatus.FAILED


# ─── Usage: Create Order Saga ─────────────────────────────
class OrderSagaOrchestrator:
    def __init__(self, inventory_client, payment_client, shipping_client):
        self.inventory = inventory_client
        self.payment = payment_client
        self.shipping = shipping_client
    
    def create_order(self, order_id: str, items: list, amount: float):
        saga = OrderSaga(order_id=order_id)
        
        # Define steps with their compensations
        saga.steps = [
            SagaStep(
                name="Reserve Inventory",
                action=lambda: self.inventory.reserve(order_id, items),
                compensation=lambda: self.inventory.release(order_id, items)
            ),
            SagaStep(
                name="Process Payment",
                action=lambda: self.payment.charge(order_id, amount),
                compensation=lambda: self.payment.refund(order_id, amount)
            ),
            SagaStep(
                name="Create Shipment",
                action=lambda: self.shipping.create(order_id),
                compensation=lambda: self.shipping.cancel(order_id)
            ),
        ]
        
        success = saga.execute()
        return saga


# ─── Example run ─────────────────────────────────────────
# orchestrator = OrderSagaOrchestrator(inventory, payment, shipping)
# saga = orchestrator.create_order("ord_123", items=["SKU1", "SKU2"], amount=99.99)
# Output:
#   Executing: Reserve Inventory
#   Executing: Process Payment
#   ❌ Failed at: Process Payment - Insufficient funds
#   🔄 Starting compensation...
#   Compensating: Reserve Inventory
#   Saga status: FAILED
```

### Java: Orchestration-Based Saga with Spring

```java
// ─── Saga Step Definition ────────────────────────────────
public class SagaStep {
    private final String name;
    private final Runnable action;
    private final Runnable compensation;
    private boolean completed = false;
    
    public SagaStep(String name, Runnable action, Runnable compensation) {
        this.name = name;
        this.action = action;
        this.compensation = compensation;
    }
    
    public void execute() {
        action.run();
        completed = true;
    }
    
    public void compensate() {
        if (completed) {
            compensation.run();
        }
    }
    
    // getters...
}

// ─── Saga Orchestrator ───────────────────────────────────
@Service
@Slf4j
public class OrderSagaOrchestrator {
    
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final ShippingClient shippingClient;
    private final SagaRepository sagaRepository;
    
    @Transactional
    public SagaResult createOrder(CreateOrderCommand command) {
        String sagaId = UUID.randomUUID().toString();
        
        List<SagaStep> steps = List.of(
            new SagaStep(
                "Reserve Inventory",
                () -> inventoryClient.reserve(command.orderId(), command.items()),
                () -> inventoryClient.release(command.orderId(), command.items())
            ),
            new SagaStep(
                "Process Payment",
                () -> paymentClient.charge(command.orderId(), command.amount()),
                () -> paymentClient.refund(command.orderId(), command.amount())
            ),
            new SagaStep(
                "Create Shipment",
                () -> shippingClient.create(command.orderId(), command.address()),
                () -> shippingClient.cancel(command.orderId())
            )
        );
        
        // Execute steps
        for (int i = 0; i < steps.size(); i++) {
            SagaStep step = steps.get(i);
            try {
                log.info("Saga {}: Executing step '{}'", sagaId, step.getName());
                step.execute();
                saveSagaState(sagaId, i, "EXECUTING");
            } catch (Exception e) {
                log.error("Saga {}: Step '{}' failed: {}", 
                    sagaId, step.getName(), e.getMessage());
                
                // Compensate all completed steps in reverse
                compensate(sagaId, steps, i);
                saveSagaState(sagaId, i, "FAILED");
                return SagaResult.failed(sagaId, step.getName(), e.getMessage());
            }
        }
        
        saveSagaState(sagaId, steps.size(), "COMPLETED");
        return SagaResult.completed(sagaId);
    }
    
    private void compensate(String sagaId, List<SagaStep> steps, int failedAt) {
        log.info("Saga {}: Starting compensation from step {}", sagaId, failedAt);
        
        for (int i = failedAt - 1; i >= 0; i--) {
            SagaStep step = steps.get(i);
            try {
                log.info("Saga {}: Compensating '{}'", sagaId, step.getName());
                step.compensate();
            } catch (Exception e) {
                // Log and mark for retry — don't stop compensation
                log.error("Saga {}: Compensation of '{}' failed, will retry", 
                    sagaId, step.getName());
                scheduleCompensationRetry(sagaId, step);
            }
        }
    }
}
```

### Choreography Example with Kafka (Python)

```python
# ─── Order Service (Producer) ─────────────────────────────
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers="kafka:9092",
    value_serializer=lambda v: json.dumps(v).encode()
)

def create_order(order_id, items, amount):
    # Save order with status PENDING
    save_order(order_id, status="PENDING")
    
    # Publish event — inventory service will react
    producer.send("order-events", value={
        "event_type": "OrderCreated",
        "order_id": order_id,
        "items": items,
        "amount": amount
    })

# ─── Inventory Service (Consumer + Producer) ──────────────
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "order-events",
    bootstrap_servers="kafka:9092",
    value_deserializer=lambda v: json.loads(v.decode())
)

for message in consumer:
    event = message.value
    
    if event["event_type"] == "OrderCreated":
        try:
            reserve_stock(event["order_id"], event["items"])
            # Success — publish next event
            producer.send("inventory-events", value={
                "event_type": "StockReserved",
                "order_id": event["order_id"],
                "amount": event["amount"]
            })
        except InsufficientStockError:
            # Failure — publish compensation event
            producer.send("order-events", value={
                "event_type": "StockReservationFailed",
                "order_id": event["order_id"]
            })
    
    elif event["event_type"] == "PaymentFailed":
        # Compensate: release previously reserved stock
        release_stock(event["order_id"])
        producer.send("inventory-events", value={
            "event_type": "StockReleased",
            "order_id": event["order_id"]
        })
```

---

## Saga Execution Guarantees

```
┌─────────────────────────────────────────────────────────────┐
│           SAGA GUARANTEES & REQUIREMENTS                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Each step is a LOCAL transaction (ACID within service)   │
│  2. Compensations must be IDEMPOTENT (safe to retry)         │
│  3. Compensations must be COMMUTATIVE (order doesn't matter  │
│     for final state)                                          │
│  4. Saga provides EVENTUAL CONSISTENCY, not strong           │
│  5. Saga does NOT provide isolation (other sagas can read    │
│     intermediate states)                                      │
│                                                               │
│  IMPORTANT:                                                   │
│  • Saga ≠ ACID transaction                                   │
│  • Intermediate states ARE visible to other operations       │
│  • Design compensations carefully — not all actions are      │
│    reversible (e.g., sending an email)                        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Handling Non-Reversible Actions (Pivot Transactions)

```
Step 1: Reserve Stock         ← Reversible (release stock)
Step 2: Charge Payment        ← Reversible (refund)
Step 3: Send Confirmation     ← NOT reversible! (email sent)
        Email                   (This is a "pivot transaction")
Step 4: Ship Package          ← Happens after pivot

RULE: Place non-reversible steps LAST, after all potentially
      failing steps have completed.
```

---

## Infrastructure — Saga with Message Broker

```
┌─────────────────────────────────────────────────────────────────┐
│                 SAGA INFRASTRUCTURE                               │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              Apache Kafka / RabbitMQ                     │     │
│  │                                                          │     │
│  │  Topics/Queues:                                          │     │
│  │    • order-events                                        │     │
│  │    • inventory-events                                    │     │
│  │    • payment-events                                      │     │
│  │    • shipping-events                                     │     │
│  │    • saga-commands (for orchestration)                   │     │
│  │    • saga-replies  (for orchestration)                   │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  Order   │  │Inventory │  │ Payment  │  │ Shipping │       │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │       │
│  ├──────────┤  ├──────────┤  ├──────────┤  ├──────────┤       │
│  │PostgreSQL│  │PostgreSQL│  │PostgreSQL│  │PostgreSQL│       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│                                                                   │
│  Optional: Saga State Store (for orchestrator)                   │
│  ┌──────────────────────────────────────────┐                   │
│  │  saga_id │ order_id │ step │ status      │                   │
│  │  s_001   │ ord_123  │  2   │ EXECUTING   │                   │
│  │  s_002   │ ord_456  │  1   │ COMPENSATING│                   │
│  └──────────────────────────────────────────┘                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Uber's Trip Saga

When you book a ride on Uber, a saga orchestrates:

```
1. Create Trip          → Trip Service (stores trip details)
2. Match Driver         → Matching Service (find nearby driver)
3. Reserve Payment      → Payment Service (hold funds)
4. Notify Driver        → Notification Service (push notification)
5. Update Map           → Maps Service (show on rider's screen)

If driver doesn't accept (step 4 fails):
  4. Compensate: Retry with next driver (or fail)
  3. Compensate: Release payment hold
  2. Compensate: Release driver reservation
  1. Compensate: Mark trip as CANCELLED
```

### Amazon Order Fulfillment

Amazon uses sagas for their order pipeline:
- Reserve inventory → Charge customer → Allocate to warehouse → Schedule shipping
- Each step has explicit compensations (release inventory, refund, etc.)
- They process millions of sagas per hour

### Airbnb Booking

```
1. Create reservation (pending)
2. Charge guest
3. Pay host (minus fee)
4. Send confirmation emails
5. Block calendar dates

If payment fails:
  → Release calendar block
  → Cancel reservation
  → Notify guest of failure
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Non-idempotent compensations | Retrying compensation creates duplicates | Make compensations idempotent (check-then-act) |
| Missing timeout handling | Saga hangs forever if a service is down | Add timeouts; treat timeout as failure |
| No saga state persistence | Orchestrator crash = saga stuck in limbo | Persist saga state to database |
| Ignoring intermediate visibility | Other operations read half-completed data | Use "semantic locks" (status: PENDING) |
| Not handling compensation failures | Compensation fails → inconsistent state forever | Retry with exponential backoff; alert on repeated failures |
| Too many steps in one saga | Complex, fragile, slow | Break into smaller sub-sagas |
| Using sagas for simple CRUD | Over-engineering | Use sagas ONLY when multiple services must coordinate |

---

## When to Use / When NOT to Use

### Use Sagas When:
- A business process spans multiple microservices
- Each service owns its own database (no shared DB)
- You need eventual consistency across services
- Compensating (undo) actions are possible for each step
- The process has a clear sequence of steps

### Do NOT Use Sagas When:
- You need **strong consistency** (all-or-nothing, immediately) → Use a single database transaction
- Operations cannot be compensated (e.g., physical shipment already dispatched)
- Simple two-service interactions → Direct API call with retry may suffice
- The process is read-only → No need for transactions at all
- You can use the **Outbox Pattern** instead (simpler for event publishing — see next chapter)

---

## Key Takeaways

1. **Sagas replace distributed transactions** — When you can't use 2PC (and you shouldn't at scale), sagas provide eventual consistency through compensating actions.

2. **Choreography = simple flows; Orchestration = complex flows** — Use choreography for 2-3 step sagas. Use orchestration when you have 4+ steps, branching logic, or need visibility.

3. **Every forward action MUST have a compensation** — If you can't undo a step, restructure the saga so that step comes last (pivot transaction).

4. **Compensations must be idempotent** — They WILL be retried. Design them to produce the same result regardless of how many times they run.

5. **Intermediate states are visible** — Unlike ACID transactions, other operations CAN see partially-completed sagas. Use status fields (PENDING, CONFIRMED) to handle this.

6. **Persist saga state** — The orchestrator must survive crashes. Store the current step, status, and context in a database.

7. **Combine with the Outbox Pattern** — For reliable event publishing within saga steps, use the Transactional Outbox (covered next) to prevent "event published but local transaction rolled back" scenarios.

---

## What's Next?

Next, we'll explore the **Outbox Pattern** — a technique for reliably publishing domain events after a local database transaction commits, solving the "dual write" problem that plagues saga implementations.

See: [05-outbox-pattern.md](./05-outbox-pattern.md)
