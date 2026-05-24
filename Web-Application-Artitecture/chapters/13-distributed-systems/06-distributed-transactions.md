# Distributed Transactions (2PC, Saga Pattern)

> **What you'll learn**: How to maintain data consistency when a single business operation spans multiple databases or microservices — including Two-Phase Commit (2PC) and the Saga pattern with their trade-offs.

---

## Real-Life Analogy: Buying a House

Buying a house involves multiple parties that must all agree:

1. **Bank** approves your mortgage
2. **Seller** agrees to the price
3. **Title company** verifies clear title
4. **Insurance company** provides homeowner's insurance

If ANY one fails, the whole deal should be cancelled. You don't want to pay for a house but not get the title, or get the title but the bank didn't actually release funds.

**Two-Phase Commit (2PC)** is like having a closing agent who:
- Phase 1: Asks everyone "Are you ready to close?" (PREPARE)
- Phase 2: If ALL say yes → "OK, everybody sign!" (COMMIT). If anyone says no → "Deal's off, everyone stand down." (ROLLBACK)

**Saga Pattern** is like doing it step-by-step with "undo" plans:
- Step 1: Bank reserves funds. (If next step fails → bank releases funds)
- Step 2: Title transfer initiated. (If next step fails → reverse title transfer)
- Step 3: Insurance activated. Done!
- Each step has a compensating action that undoes it if a later step fails.

---

## Core Concept Explained Step-by-Step

### The Problem: Cross-Service Consistency

```
SCENARIO: E-commerce order placement
═══════════════════════════════════════════════════

When a user clicks "Buy Now," THREE things must happen:

1. Payment Service → Charge $100
2. Inventory Service → Reserve 1 item
3. Order Service → Create order record

ALL THREE must succeed, or ALL THREE must fail.

What if Payment succeeds but Inventory fails?
─────────────────────────────────────────────────
Payment: -$100 ✅ (money taken)
Inventory: "Out of stock!" ❌
Order: never created ❌

Result: Customer lost $100 but got no item! 💀
THIS IS UNACCEPTABLE.
```

### Why Regular Transactions Don't Work

```
SINGLE DATABASE: ACID transaction (easy!)
═══════════════════════════════════════════════════

BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100;
  UPDATE inventory SET quantity = quantity - 1;
  INSERT INTO orders (user_id, amount) VALUES (1, 100);
COMMIT;  -- All or nothing! Database handles it.

MULTIPLE DATABASES/SERVICES: No shared transaction! 
═══════════════════════════════════════════════════

┌──────────┐     ┌───────────────┐     ┌────────────┐
│ Payment  │     │   Inventory   │     │   Order    │
│ Service  │     │   Service     │     │  Service   │
│(Stripe DB)│     │(MongoDB)      │     │(PostgreSQL)│
└──────────┘     └───────────────┘     └────────────┘
     │                  │                     │
  Own DB            Own DB                Own DB
  Own tx            Own tx                Own tx

No single "BEGIN TRANSACTION" spans all three!
Each has its own independent database.
```

---

## Solution 1: Two-Phase Commit (2PC)

### How 2PC Works

```
TWO-PHASE COMMIT:
═══════════════════════════════════════════════════

COORDINATOR (Transaction Manager)
      │
      │         PHASE 1: PREPARE (Can you commit?)
      │
      ├──── "Prepare?" ──▶ Participant A (Payment)
      │                         → Locks resources
      │                         → Writes to WAL
      │                         → Responds: "YES, ready" ✅
      │
      ├──── "Prepare?" ──▶ Participant B (Inventory)
      │                         → Locks resources
      │                         → Writes to WAL
      │                         → Responds: "YES, ready" ✅
      │
      ├──── "Prepare?" ──▶ Participant C (Orders)
      │                         → Locks resources
      │                         → Writes to WAL
      │                         → Responds: "YES, ready" ✅
      │
      │         ALL said YES!
      │
      │         PHASE 2: COMMIT (Go ahead!)
      │
      ├──── "Commit!" ──▶ Participant A → Commits ✅
      ├──── "Commit!" ──▶ Participant B → Commits ✅
      └──── "Commit!" ──▶ Participant C → Commits ✅

DONE! All three committed atomically.
```

### 2PC Failure Scenarios

```
SCENARIO 1: Participant says NO in Phase 1
═══════════════════════════════════════════════════

Coordinator: "Prepare?"
  Payment: "YES" ✅
  Inventory: "NO — out of stock!" ❌
  Orders: "YES" ✅

Coordinator: "ABORT everything!"
  Payment: rolls back (refund)
  Inventory: nothing to undo
  Orders: rolls back (delete order)

SAFE! ✅

SCENARIO 2: Coordinator crashes AFTER Phase 1, BEFORE Phase 2
═══════════════════════════════════════════════════

Coordinator: "Prepare?"
  All participants: "YES" ✅
  
Coordinator: 💀 CRASHES before sending Commit/Abort

Participants are now STUCK:
  - They promised to commit (locked resources)
  - They can't commit (no instruction from coordinator)
  - They can't abort (coordinator might come back saying "commit")
  - Resources remain LOCKED indefinitely!

THIS IS THE 2PC BLOCKING PROBLEM! 🔥

SCENARIO 3: Participant crashes AFTER voting YES
═══════════════════════════════════════════════════

Coordinator: "Prepare?"
  All: "YES" ✅
  
Coordinator: "Commit!"
  Payment: Commits ✅
  Inventory: 💀 CRASHES before committing
  Orders: Commits ✅
  
When Inventory recovers:
  → Checks its WAL (Write-Ahead Log)
  → Sees it voted YES for this transaction
  → Applies the commit (because it promised)
  
SAFE (assuming WAL survives) ✅
```

### 2PC Problems

```
┌──────────────────────────────────────────────────────────────┐
│                    2PC PROBLEMS                               │
├─────────────────────┬────────────────────────────────────────┤
│ Blocking            │ If coordinator dies, participants are  │
│                     │ stuck holding locks indefinitely       │
├─────────────────────┼────────────────────────────────────────┤
│ Performance         │ All participants locked during both    │
│                     │ phases — high latency, low throughput  │
├─────────────────────┼────────────────────────────────────────┤
│ Single point of     │ Coordinator is a bottleneck and       │
│ failure             │ single point of failure               │
├─────────────────────┼────────────────────────────────────────┤
│ Not partition-      │ Network partition between coordinator  │
│ tolerant            │ and participant = deadlock             │
├─────────────────────┼────────────────────────────────────────┤
│ Latency             │ 2 round-trips minimum (Prepare +      │
│                     │ Commit), often cross-network           │
└─────────────────────┴────────────────────────────────────────┘
```

---

## Solution 2: Saga Pattern

### How Sagas Work

Instead of locking everything, a Saga executes steps **sequentially** with **compensating transactions** for rollback:

```
SAGA: Forward steps + Compensating actions
═══════════════════════════════════════════════════

Forward Flow (happy path):
T1: Payment.charge($100) ──▶ T2: Inventory.reserve(1) ──▶ T3: Order.create()
                                                                    │
                                                                 SUCCESS!

Compensating Flow (when T3 fails):
T3 fails! ──▶ C2: Inventory.release(1) ──▶ C1: Payment.refund($100)
                                                    │
                                                 ROLLED BACK!

Each step Ti has a compensating step Ci:
┌──────┬─────────────────────┬───────────────────────────┐
│ Step │ Forward Action       │ Compensating Action        │
├──────┼─────────────────────┼───────────────────────────┤
│  T1  │ Charge payment       │ C1: Refund payment         │
│  T2  │ Reserve inventory    │ C2: Release inventory      │
│  T3  │ Create order         │ C3: Cancel order           │
│  T4  │ Send confirmation    │ C4: Send cancellation email│
└──────┴─────────────────────┴───────────────────────────┘
```

### Two Saga Approaches

```
APPROACH 1: CHOREOGRAPHY (Event-driven)
═══════════════════════════════════════════════════

Each service listens for events and acts:

Payment Service ──"PaymentCompleted"──▶ Event Bus
                                            │
Inventory Service ◀── listens ──────────────┘
        │
        ├── "InventoryReserved" ──▶ Event Bus
                                        │
Order Service ◀── listens ──────────────┘
        │
        └── "OrderCreated" ──▶ Event Bus ──▶ Done!

If Inventory fails:
  Publishes "InventoryFailed" → Payment listens → Refunds

Pros: No single orchestrator, loosely coupled
Cons: Hard to track, complex failure handling


APPROACH 2: ORCHESTRATION (Central coordinator)
═══════════════════════════════════════════════════

Saga Orchestrator controls the flow:

┌────────────────────┐
│  Saga Orchestrator │
│  (Order Saga)      │
└────────┬───────────┘
         │
         ├── 1. "Charge $100" ──▶ Payment Service ──▶ "OK" ✅
         │
         ├── 2. "Reserve item" ──▶ Inventory Service ──▶ "OK" ✅
         │
         ├── 3. "Create order" ──▶ Order Service ──▶ "FAILED" ❌
         │
         │   FAILURE! Execute compensations:
         │
         ├── C2. "Release item" ──▶ Inventory Service ──▶ "OK" ✅
         │
         └── C1. "Refund $100" ──▶ Payment Service ──▶ "OK" ✅

Pros: Clear flow, easy to debug, centralized logic
Cons: Orchestrator is a single point, tighter coupling
```

---

## How It Works Internally

### 2PC Protocol Messages

```
2PC MESSAGE FLOW (detailed):
═══════════════════════════════════════════════════

Coordinator                    Participant
     │                              │
     │──── PREPARE ────────────────▶│
     │                              │ Write to WAL
     │                              │ Acquire locks
     │◀─── VOTE_YES ───────────────│
     │                              │
     │ (collect all votes)          │
     │                              │
     │──── GLOBAL_COMMIT ──────────▶│
     │                              │ Apply changes
     │                              │ Release locks
     │◀─── ACK ────────────────────│
     │                              │

WAL (Write-Ahead Log) entries:
─────────────────────────────────────────
Coordinator WAL:
  [START txn-123]
  [PREPARE txn-123 → A, B, C]
  [VOTE_YES from A]
  [VOTE_YES from B]  
  [VOTE_YES from C]
  [DECISION: COMMIT txn-123]  ← Point of no return!
  [ACK from A]
  [ACK from B]
  [ACK from C]
  [END txn-123]

If coordinator crashes and recovers:
  → Read WAL → see DECISION: COMMIT → resend COMMIT to all
```

### Saga State Machine

```
SAGA STATE MACHINE:
═══════════════════════════════════════════════════

          ┌───────────┐
          │  STARTED  │
          └─────┬─────┘
                │ Execute T1
                ▼
          ┌───────────┐      ┌───────────────┐
          │ T1_DONE   │─fail─▶│ COMPENSATING │
          └─────┬─────┘      └───────┬───────┘
                │ Execute T2          │ Execute C1
                ▼                     ▼
          ┌───────────┐      ┌───────────────┐
          │ T2_DONE   │─fail─▶│ C1_DONE      │
          └─────┬─────┘      └───────┬───────┘
                │ Execute T3          │
                ▼                     ▼
          ┌───────────┐      ┌───────────────┐
          │ T3_DONE   │      │  COMPENSATED  │
          └─────┬─────┘      │  (Rolled Back)│
                │             └───────────────┘
                ▼
          ┌───────────┐
          │ COMPLETED │
          │ (Success) │
          └───────────┘

Each state is persisted — if orchestrator crashes,
it can recover and resume from the last saved state.
```

---

## Code Examples

### Python: Saga Orchestrator

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Callable, List, Optional
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("saga")

class SagaStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    FAILED = "failed"

@dataclass
class SagaStep:
    name: str
    action: Callable  # Forward action
    compensation: Callable  # Undo action
    completed: bool = False

class SagaOrchestrator:
    """Orchestrates a saga with forward actions and compensations."""
    
    def __init__(self, saga_id: str):
        self.saga_id = saga_id
        self.steps: List[SagaStep] = []
        self.status = SagaStatus.PENDING
        self.completed_steps: List[SagaStep] = []
    
    def add_step(self, name: str, action: Callable, compensation: Callable):
        """Add a step with its compensating action."""
        self.steps.append(SagaStep(name=name, action=action, compensation=compensation))
        return self  # Fluent API
    
    def execute(self) -> bool:
        """Execute the saga. Returns True if successful, False if compensated."""
        self.status = SagaStatus.RUNNING
        logger.info(f"[Saga {self.saga_id}] Starting execution")
        
        for step in self.steps:
            try:
                logger.info(f"[Saga {self.saga_id}] Executing: {step.name}")
                step.action()
                step.completed = True
                self.completed_steps.append(step)
                logger.info(f"[Saga {self.saga_id}] ✅ {step.name} succeeded")
                
            except Exception as e:
                logger.error(f"[Saga {self.saga_id}] ❌ {step.name} failed: {e}")
                self._compensate()
                return False
        
        self.status = SagaStatus.COMPLETED
        logger.info(f"[Saga {self.saga_id}] 🎉 Saga completed successfully!")
        return True
    
    def _compensate(self):
        """Execute compensating actions in reverse order."""
        self.status = SagaStatus.COMPENSATING
        logger.info(f"[Saga {self.saga_id}] Starting compensation...")
        
        for step in reversed(self.completed_steps):
            try:
                logger.info(f"[Saga {self.saga_id}] Compensating: {step.name}")
                step.compensation()
                logger.info(f"[Saga {self.saga_id}] ↩️  {step.name} compensated")
            except Exception as e:
                # Compensation failed! This needs manual intervention.
                logger.critical(
                    f"[Saga {self.saga_id}] 🔥 Compensation for {step.name} FAILED: {e}"
                    " — MANUAL INTERVENTION REQUIRED"
                )
        
        self.status = SagaStatus.FAILED


# --- Example: Order Placement Saga ---

class PaymentService:
    def charge(self, user_id: str, amount: float):
        logger.info(f"  Payment: Charging user {user_id} ${amount}")
        # In real code: call Stripe API
        if amount > 10000:
            raise Exception("Amount exceeds limit")
    
    def refund(self, user_id: str, amount: float):
        logger.info(f"  Payment: Refunding user {user_id} ${amount}")

class InventoryService:
    def __init__(self):
        self.stock = {"ITEM-001": 5}
    
    def reserve(self, item_id: str, qty: int):
        if self.stock.get(item_id, 0) < qty:
            raise Exception(f"Insufficient stock for {item_id}")
        self.stock[item_id] -= qty
        logger.info(f"  Inventory: Reserved {qty}x {item_id}")
    
    def release(self, item_id: str, qty: int):
        self.stock[item_id] = self.stock.get(item_id, 0) + qty
        logger.info(f"  Inventory: Released {qty}x {item_id}")

class OrderService:
    def create(self, user_id: str, item_id: str):
        logger.info(f"  Order: Created order for {user_id}")
    
    def cancel(self, user_id: str, item_id: str):
        logger.info(f"  Order: Cancelled order for {user_id}")


# Execute the saga
payment = PaymentService()
inventory = InventoryService()
orders = OrderService()

saga = SagaOrchestrator("ORDER-001")
saga.add_step(
    name="Charge Payment",
    action=lambda: payment.charge("user-123", 99.99),
    compensation=lambda: payment.refund("user-123", 99.99)
).add_step(
    name="Reserve Inventory",
    action=lambda: inventory.reserve("ITEM-001", 1),
    compensation=lambda: inventory.release("ITEM-001", 1)
).add_step(
    name="Create Order",
    action=lambda: orders.create("user-123", "ITEM-001"),
    compensation=lambda: orders.cancel("user-123", "ITEM-001")
)

result = saga.execute()
print(f"\nSaga result: {'SUCCESS' if result else 'COMPENSATED'}")
```

### Java: Two-Phase Commit Coordinator

```java
import java.util.*;
import java.util.concurrent.*;

/**
 * Simplified Two-Phase Commit coordinator.
 * In production, use frameworks like Atomikos, Narayana, or Seata.
 */
public class TwoPhaseCommit {
    
    interface Participant {
        String getName();
        boolean prepare(String transactionId);  // Phase 1
        void commit(String transactionId);      // Phase 2 (success)
        void rollback(String transactionId);    // Phase 2 (failure)
    }
    
    static class Coordinator {
        private final List<Participant> participants = new ArrayList<>();
        private final Map<String, String> transactionLog = new ConcurrentHashMap<>();
        
        public void registerParticipant(Participant p) {
            participants.add(p);
        }
        
        public boolean executeTransaction(String txnId) {
            transactionLog.put(txnId, "STARTED");
            System.out.printf("[Coordinator] Starting transaction: %s%n", txnId);
            
            // ═══ PHASE 1: PREPARE ═══
            System.out.println("\n--- PHASE 1: PREPARE ---");
            List<Participant> votedYes = new ArrayList<>();
            boolean allReady = true;
            
            for (Participant p : participants) {
                System.out.printf("[Coordinator] Asking %s to prepare...%n", p.getName());
                boolean ready = p.prepare(txnId);
                
                if (ready) {
                    votedYes.add(p);
                    System.out.printf("[Coordinator] %s voted YES ✅%n", p.getName());
                } else {
                    System.out.printf("[Coordinator] %s voted NO ❌%n", p.getName());
                    allReady = false;
                    break;
                }
            }
            
            // ═══ PHASE 2: COMMIT or ROLLBACK ═══
            if (allReady) {
                System.out.println("\n--- PHASE 2: COMMIT ---");
                transactionLog.put(txnId, "COMMITTING");
                
                for (Participant p : participants) {
                    p.commit(txnId);
                    System.out.printf("[Coordinator] %s committed ✅%n", p.getName());
                }
                
                transactionLog.put(txnId, "COMMITTED");
                System.out.printf("%n[Coordinator] Transaction %s COMMITTED 🎉%n", txnId);
                return true;
                
            } else {
                System.out.println("\n--- PHASE 2: ROLLBACK ---");
                transactionLog.put(txnId, "ABORTING");
                
                for (Participant p : votedYes) {
                    p.rollback(txnId);
                    System.out.printf("[Coordinator] %s rolled back ↩️%n", p.getName());
                }
                
                transactionLog.put(txnId, "ABORTED");
                System.out.printf("%n[Coordinator] Transaction %s ABORTED ❌%n", txnId);
                return false;
            }
        }
    }
    
    // --- Example Participants ---
    
    static class PaymentParticipant implements Participant {
        private double balance = 1000;
        private final Map<String, Double> pending = new HashMap<>();
        
        public String getName() { return "PaymentService"; }
        
        public boolean prepare(String txnId) {
            double amount = 100;  // Simplified
            if (balance >= amount) {
                balance -= amount;
                pending.put(txnId, amount);
                return true;  // "I can commit this"
            }
            return false;  // "Insufficient funds"
        }
        
        public void commit(String txnId) {
            pending.remove(txnId);  // Finalize deduction
        }
        
        public void rollback(String txnId) {
            Double amount = pending.remove(txnId);
            if (amount != null) balance += amount;  // Restore balance
        }
    }
    
    static class InventoryParticipant implements Participant {
        private int stock = 5;
        private final Map<String, Integer> reserved = new HashMap<>();
        
        public String getName() { return "InventoryService"; }
        
        public boolean prepare(String txnId) {
            if (stock > 0) {
                stock--;
                reserved.put(txnId, 1);
                return true;
            }
            return false;  // "Out of stock"
        }
        
        public void commit(String txnId) {
            reserved.remove(txnId);  // Finalize reservation
        }
        
        public void rollback(String txnId) {
            Integer qty = reserved.remove(txnId);
            if (qty != null) stock += qty;  // Release reservation
        }
    }
    
    public static void main(String[] args) {
        Coordinator coordinator = new Coordinator();
        coordinator.registerParticipant(new PaymentParticipant());
        coordinator.registerParticipant(new InventoryParticipant());
        
        // Successful transaction
        coordinator.executeTransaction("TXN-001");
    }
}
```

---

## Infrastructure Examples

### Saga with Kafka Events

```
ORDER SAGA WITH KAFKA (Choreography):
═══════════════════════════════════════════════════

Topics:
  - orders.created
  - payments.charged
  - payments.failed
  - inventory.reserved
  - inventory.failed
  - orders.completed
  - orders.cancelled

Flow:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Order Service ──publish──▶ [orders.created]                │
│                                     │                        │
│  Payment Service ◀── subscribes ────┘                        │
│       │                                                      │
│       ├─ success ──▶ [payments.charged]                     │
│       │                     │                                │
│       │  Inventory ◀────────┘                                │
│       │       │                                              │
│       │       ├─ success ──▶ [inventory.reserved]           │
│       │       │                     │                        │
│       │       │  Order Service ◀────┘                        │
│       │       │       └──▶ [orders.completed] ✅             │
│       │       │                                              │
│       │       └─ failure ──▶ [inventory.failed]             │
│       │                           │                          │
│       │  Payment Service ◀────────┘                          │
│       │       └── refund ──▶ [payments.refunded]            │
│       │                                                      │
│       └─ failure ──▶ [payments.failed]                      │
│                           │                                  │
│  Order Service ◀──────────┘                                  │
│       └── cancel ──▶ [orders.cancelled] ❌                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Seata: Distributed Transaction Framework

```
SEATA (used by Alibaba for 11.11 sales):
═══════════════════════════════════════════════════

Architecture:
┌────────────────────────────────────────────────┐
│            Transaction Coordinator (TC)         │
│         (manages global transactions)          │
└─────────┬──────────────────┬───────────────────┘
          │                  │
    ┌─────▼─────┐     ┌─────▼─────┐
    │  TM (App) │     │ RM (DB)   │
    │ (begins/  │     │ (manages  │
    │  commits) │     │  branches)│
    └───────────┘     └───────────┘

Modes:
- AT (Auto Transaction): SQL-level auto rollback
- TCC (Try-Confirm-Cancel): Business-level 2PC
- Saga: Long-running compensating transactions
- XA: Standard 2PC protocol
```

---

## 2PC vs Saga Comparison

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│                  │      2PC             │      SAGA            │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Consistency      │ Strong (ACID)        │ Eventual             │
│ Isolation        │ Full isolation       │ No isolation*        │
│ Blocking         │ YES (locks held)     │ NO (no locks)        │
│ Performance      │ Slow (sync, locks)   │ Fast (async steps)   │
│ Failure handling │ Automatic rollback   │ Compensating actions │
│ Complexity       │ Protocol is simple   │ Compensation logic   │
│ Use case         │ Small, fast txns     │ Long-running, cross- │
│                  │                      │ service operations   │
│ Scalability      │ Poor (blocking)      │ Good (no blocking)   │
│ Availability     │ Reduced (coordinator)│ High (no coordinator†)│
└──────────────────┴──────────────────────┴──────────────────────┘

* Saga isolation issue: intermediate states are visible.
  Between T1 and T2, another process can see the partially
  completed state. Solutions: semantic locks, versioning.

† Choreography sagas have no coordinator; orchestration sagas do.
```

---

## Real-World Example

### Uber: Saga for Ride Completion

```
Uber's ride completion saga:
═══════════════════════════════════════════════════

When a ride ends, multiple services must coordinate:

T1: Trip Service → Mark ride as completed
T2: Pricing Service → Calculate final fare
T3: Payment Service → Charge rider
T4: Payment Service → Pay driver
T5: Rating Service → Prompt for rating
T6: Notification Service → Send receipt

If T3 (charge rider) fails:
  C2: Pricing → void fare calculation
  C1: Trip → revert completion status
  Alert: notify support team

Uber uses orchestration saga (Cadence/Temporal workflow engine):
- Durable execution (survives crashes)
- Automatic retries with exponential backoff
- Full visibility into saga state
- Timeouts on each step
```

### Amazon: 2PC Within, Sagas Across

```
Amazon's approach:
═══════════════════════════════════════════════════

WITHIN a single service (e.g., Inventory):
  → Use local database transaction (ACID)
  → 2PC if multiple tables in same DB cluster

ACROSS services:
  → Saga pattern (no distributed transactions!)
  → Events via SQS/SNS for choreography
  → Step Functions for orchestration

"We tried distributed transactions early on.
 They don't scale. Sagas with idempotent 
 compensations are the way." — Amazon engineers

Key insight:
  The "eventually consistent" window is usually
  < 1 second. Users don't notice.
  
  But the performance gain from NOT blocking
  is enormous at Amazon's scale.
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using 2PC across microservices | Blocking + performance + availability issues at scale | Use Saga pattern for cross-service transactions |
| Non-idempotent compensations | If compensation runs twice (retry), it causes more problems | Make all compensations idempotent |
| No saga state persistence | If orchestrator crashes, saga state is lost mid-execution | Persist saga state after each step (database/event log) |
| Ignoring intermediate state visibility | Other services see partial saga execution | Use semantic locks or version flags |
| No timeout on saga steps | A hung service blocks the entire saga forever | Add timeouts + dead letter queues for stuck steps |
| Forgetting to handle compensation failure | What if the refund call fails? | Alert + manual queue for human resolution |

---

## When to Use / When NOT to Use

### Use 2PC When:
- ✅ Short-lived transactions (< 1 second)
- ✅ All participants are within the same data center
- ✅ You absolutely need ACID guarantees
- ✅ Limited number of participants (2-3)
- ✅ Database-level transactions (XA-compatible databases)

### Use Saga When:
- ✅ Long-running business processes (seconds to days)
- ✅ Cross-service/cross-region operations
- ✅ High availability is required
- ✅ Many services involved (4+)
- ✅ Eventual consistency is acceptable

### Use Neither When:
- ✅ Everything fits in one database → just use a local transaction!
- ✅ Operations are naturally idempotent → just retry on failure
- ✅ You can redesign to avoid cross-service writes

---

## Key Takeaways

- 🔑 **The problem**: When a business operation spans multiple services/databases, you can't use a single ACID transaction.
- 🔑 **2PC** provides strong consistency but is blocking, slow, and fragile. Use for short, local transactions only.
- 🔑 **Saga** provides eventual consistency with compensating actions. Non-blocking, scalable, but more complex to implement.
- 🔑 **Choreography sagas** (events) are loosely coupled but hard to track. **Orchestration sagas** are centralized but easier to debug.
- 🔑 **Every saga step must be idempotent** — it might be executed more than once due to retries.
- 🔑 **Persist saga state** — if the orchestrator crashes, it must resume where it left off.
- 🔑 **In practice**: Use local transactions within services, Sagas across services. This is what Amazon, Uber, and Netflix do.

---

## What's Next?

Distributed transactions need a leader to coordinate them — but who decides who the leader is? The next chapter covers **[Leader Election — Who's the Boss?](./07-leader-election.md)** — how distributed systems choose a single coordinator node safely.
