# Consistency Models (Strong, Eventual, Causal)

> **What you'll learn**: The different levels of consistency guarantees in distributed systems — from the strictest (linearizability) to the most relaxed (eventual consistency) — when each is appropriate, and how they work under the hood.

---

## Real-Life Analogy: The Library Book Catalog

Imagine a library with multiple branches across a city, each with its own copy of the book catalog:

- **Strong Consistency**: When the main branch adds a new book, ALL branches update their catalogs INSTANTLY before anyone can check. If you visit any branch, you always see the complete, up-to-date catalog. *(Slow — everyone waits.)*

- **Eventual Consistency**: When a book is added, it shows up at the main branch immediately. Other branches get the update "eventually" — maybe in minutes, maybe hours. If you check two branches right now, they might show different catalogs. *(Fast — but confusing sometimes.)*

- **Causal Consistency**: If Branch A adds Book X, and then Branch B adds a review of Book X — everyone is guaranteed to see Book X BEFORE the review. You'll never see a review for a book that doesn't exist yet. But you might not see the LATEST books immediately. *(A middle ground — respects cause and effect.)*

---

## Core Concept Explained Step-by-Step

### The Consistency Spectrum

```
STRONGEST ◄─────────────────────────────────────────────► WEAKEST
                                                           
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│Linearizable  │ Sequential   │   Causal     │  Read-Your-  │  Eventual    │
│(Strictest)   │ Consistency  │ Consistency  │  Own-Writes  │ Consistency  │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│Every op      │Global order  │Causally      │You see YOUR  │Eventually    │
│appears       │exists, but   │related ops   │own writes    │all nodes     │
│instant &     │real-time     │are ordered;  │immediately;  │converge to   │
│atomic to all │order not     │concurrent    │others may    │same value    │
│observers     │guaranteed    │ops unordered │see stale     │(no time      │
│              │              │              │data          │guarantee)    │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│Slowest       │              │              │              │Fastest       │
│Most expensive│              │              │              │Cheapest      │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

### 1. Strong Consistency (Linearizability)

The **gold standard**. Once a write completes, ALL subsequent reads from ANY node return that write.

```
LINEARIZABILITY:
═══════════════════════════════════════════════════════════════

Timeline (real time flows left to right →)
─────────────────────────────────────────────────────────────

Writer:     ├── write(x=1) ──┤         ├── write(x=2) ──┤
                              │                           │
Reader A:              ├── read(x) ──┤                    │
                       Returns: 1  ✅                     │
                                                          │
Reader B:                                    ├── read(x) ──┤
                                             Returns: 2  ✅

GUARANTEE: If write(x=1) finishes BEFORE read(x) starts,
           the read MUST return 1 (or newer).
           No reader ever sees an "old" value after a write completes.
```

**How it works internally:**
- All writes go through a single leader or consensus protocol
- Reads must either go to the leader or verify with a quorum
- Expensive: requires coordination on EVERY operation

### 2. Sequential Consistency

A global order of operations exists, and all nodes see the SAME order — but this order doesn't have to match real-time clock order.

```
SEQUENTIAL CONSISTENCY:
═══════════════════════════════════════════════════════════════

Process 1: write(x=A)     write(y=B)
Process 2:          write(x=C)     read(y)

Valid sequential order (one possible interleaving):
  write(x=A) → write(x=C) → write(y=B) → read(y)=B  ✅

Another valid order:
  write(x=C) → write(x=A) → write(y=B) → read(y)=B  ✅

Invalid: reading y=B before write(y=B) in the sequence
  read(y)=B → write(y=B)  ❌

Key: All processes AGREE on one order (even if it differs from wall-clock time)
```

### 3. Causal Consistency

If operation A **causes** operation B, then everyone sees A before B. But operations with no causal relationship can appear in any order.

```
CAUSAL CONSISTENCY:
═══════════════════════════════════════════════════════════════

User Alice: posts "Hello!"                    (Event A)
User Bob:   sees "Hello!", replies "Hi back!" (Event B)
            B CAUSALLY DEPENDS on A (Bob saw Alice's post first)

GUARANTEE: Everyone sees "Hello!" before "Hi back!"
           (causal order preserved)

User Carol (on different server): posts "Nice weather" (Event C)
           C has NO causal relation to A or B

ALLOWED: Different users may see Carol's post in any position:
  Observer 1: Hello! → Nice weather → Hi back!    ✅
  Observer 2: Hello! → Hi back! → Nice weather    ✅
  Observer 3: Nice weather → Hello! → Hi back!    ✅
  
NOT ALLOWED: Hi back! → Hello!  ❌ (violates causal order)
```

### 4. Read-Your-Own-Writes (Session Consistency)

You always see your own writes immediately — but other users might see stale data.

```
READ-YOUR-OWN-WRITES:
═══════════════════════════════════════════════════════════════

You update your profile name to "Alice Smith"
You immediately reload the page → you see "Alice Smith" ✅

Meanwhile:
Bob visits your profile → might still see "Alice Jones" 
                          (old name, for a few seconds)

This is weaker than strong consistency, but much more practical.
Most users don't notice OTHER people's stale data —
they DO notice their OWN stale data ("I just changed this!?")
```

### 5. Eventual Consistency

The **weakest useful guarantee**. If no new writes occur, all replicas WILL converge to the same value... eventually.

```
EVENTUAL CONSISTENCY:
═══════════════════════════════════════════════════════════════

Time 0:  Write x=42 on Node A
         Node A: x=42
         Node B: x=??? (hasn't received update yet)
         Node C: x=??? (hasn't received update yet)

Time 1:  Replication in progress...
         Node A: x=42
         Node B: x=42  ← received!
         Node C: x=??? ← still waiting

Time 2:  All caught up
         Node A: x=42
         Node B: x=42
         Node C: x=42  ← finally consistent!

GUARANTEE: "Eventually" all nodes agree.
NO GUARANTEE: How long "eventually" takes.
              (Could be milliseconds, could be seconds)
```

---

## How It Works Internally

### Implementing Strong Consistency

```
APPROACH 1: Single Leader (simplest)
═══════════════════════════════════════════════════

All writes/reads go through ONE node:

Client ──write──▶ ┌────────────┐ ──replicate──▶ Follower 1
Client ──read───▶ │   LEADER   │ ──replicate──▶ Follower 2
                  └────────────┘ ──replicate──▶ Follower 3

Pros: Simple, always consistent
Cons: Leader is bottleneck + single point of failure

APPROACH 2: Quorum Reads/Writes
═══════════════════════════════════════════════════

W + R > N (ensures overlap between write & read nodes)

Write with W=2 (of N=3):
  Client → Node1 ✅, Node2 ✅, Node3 ❌ → Success

Read with R=2 (of N=3):
  Client ← Node1 (v2), Node2 (v2), Node3 (v1-stale)
  At least one node in read set has latest → return v2

APPROACH 3: Consensus Protocol (Raft/Paxos)
═══════════════════════════════════════════════════

Nodes VOTE on every write:
  1. Leader proposes write
  2. Majority must agree
  3. Only then is write committed
  4. Reads from leader are always fresh
  
(See Chapter 13.5 for details on Raft and Paxos)
```

### Implementing Eventual Consistency

```
ANTI-ENTROPY REPAIR:
═══════════════════════════════════════════════════

Nodes periodically compare their data and fix differences:

Node A: {x:1, y:2, z:3}
Node B: {x:1, y:2}        ← missing z!

Repair: Node A → Node B: "Here's z:3, you're missing it"
Node B: {x:1, y:2, z:3}   ← fixed!

Methods:
1. Merkle Trees — efficiently find differences
2. Read Repair — fix stale data when read
3. Hinted Handoff — store writes for downed nodes

CONFLICT RESOLUTION (when two nodes have different values):
═══════════════════════════════════════════════════

Node A: x = "hello" (written at timestamp 100)
Node B: x = "world" (written at timestamp 102)

Resolution strategies:
┌─────────────────────┬──────────────────────────────────┐
│ Last-Write-Wins     │ Highest timestamp wins (x="world")│
│ (LWW)              │ Simple but can lose data          │
├─────────────────────┼──────────────────────────────────┤
│ Multi-Value         │ Keep both: x=["hello","world"]   │
│ (Siblings)          │ Let application resolve          │
├─────────────────────┼──────────────────────────────────┤
│ CRDTs              │ Merge automatically (counters,    │
│                    │ sets) — no conflicts possible     │
├─────────────────────┼──────────────────────────────────┤
│ Vector Clocks      │ Detect conflicts; track causality │
│                    │ (See Chapter 13.9)                │
└─────────────────────┴──────────────────────────────────┘
```

### Implementing Causal Consistency

```
TRACKING CAUSALITY WITH DEPENDENCY VECTORS:
═══════════════════════════════════════════════════

Each operation carries metadata about what it "saw":

Operation 1: write(x=1) → dependency: {}
Operation 2: read(x)=1, then write(y=2) → dependency: {op1}
Operation 3: read(y)=2, then write(z=3) → dependency: {op1, op2}

When Node B receives Operation 3:
  Check: "Do I have op1 and op2?"
  If YES → apply op3
  If NO → WAIT until op1 and op2 arrive first

This ensures causal order is preserved without full synchronization.

Visual:
─────────────────────────────────────
Node A: write(x=1) ─────────────────────────────▶
             │                                    
             ▼ (read x=1)                         
Node B:          write(y=2) ────────────────────▶
                      │
                      ▼ (read y=2)
Node C:                   write(z=3) ───────────▶

Everyone sees: x=1 → y=2 → z=3 (causal order)
```

---

## Code Examples

### Python: Demonstrating Consistency Levels

```python
import time
import threading
import random
from collections import defaultdict
from typing import Dict, List, Optional, Any

class ReplicatedStore:
    """Demonstrates different consistency models."""
    
    def __init__(self, num_replicas: int = 3, replication_delay: float = 0.1):
        self.replicas: List[Dict[str, Any]] = [{} for _ in range(num_replicas)]
        self.replication_delay = replication_delay
        self.version_clock = 0
        self.lock = threading.Lock()
    
    # ─── STRONG CONSISTENCY (Linearizable) ─────────────────────
    def strong_write(self, key: str, value: Any) -> bool:
        """Write to ALL replicas synchronously before returning."""
        with self.lock:
            self.version_clock += 1
            versioned_value = {"value": value, "version": self.version_clock}
            
            # Synchronous replication — wait for ALL
            for replica in self.replicas:
                time.sleep(self.replication_delay)  # Simulate network
                replica[key] = versioned_value
            
            return True  # Only returns after ALL replicas confirm
    
    def strong_read(self, key: str) -> Optional[Any]:
        """Read from quorum to guarantee freshness."""
        responses = []
        for replica in self.replicas:
            if key in replica:
                responses.append(replica[key])
        
        if not responses:
            return None
        
        # Return highest version (guaranteed to be latest due to quorum)
        latest = max(responses, key=lambda r: r["version"])
        return latest["value"]
    
    # ─── EVENTUAL CONSISTENCY ──────────────────────────────────
    def eventual_write(self, key: str, value: Any) -> bool:
        """Write to primary only; replicate asynchronously."""
        with self.lock:
            self.version_clock += 1
            versioned_value = {"value": value, "version": self.version_clock}
        
        # Write to primary immediately
        self.replicas[0][key] = versioned_value
        
        # Async replication (fire and forget)
        def replicate():
            for i in range(1, len(self.replicas)):
                time.sleep(self.replication_delay)
                self.replicas[i][key] = versioned_value
        
        threading.Thread(target=replicate, daemon=True).start()
        return True  # Returns immediately!
    
    def eventual_read(self, key: str, replica_id: int = -1) -> Optional[Any]:
        """Read from any replica (might be stale)."""
        if replica_id == -1:
            replica_id = random.randint(0, len(self.replicas) - 1)
        
        if key in self.replicas[replica_id]:
            return self.replicas[replica_id][key]["value"]
        return None
    
    # ─── CAUSAL CONSISTENCY ────────────────────────────────────
    def causal_write(self, key: str, value: Any, 
                     depends_on: List[int] = None) -> int:
        """Write with causal dependencies tracked."""
        with self.lock:
            self.version_clock += 1
            op_id = self.version_clock
        
        versioned_value = {
            "value": value,
            "version": op_id,
            "depends_on": depends_on or []
        }
        
        # Apply to primary
        self.replicas[0][key] = versioned_value
        
        # Replicate with dependency info
        def replicate_causal():
            for i in range(1, len(self.replicas)):
                time.sleep(self.replication_delay)
                # In real system: check dependencies are satisfied first
                self.replicas[i][key] = versioned_value
        
        threading.Thread(target=replicate_causal, daemon=True).start()
        return op_id


# --- Demonstration ---
store = ReplicatedStore(num_replicas=3, replication_delay=0.05)

print("=== Strong Consistency ===")
start = time.time()
store.strong_write("balance", 1000)
print(f"Write took: {(time.time()-start)*1000:.1f}ms")
print(f"Any replica reads: {store.strong_read('balance')}")

print("\n=== Eventual Consistency ===")
start = time.time()
store.eventual_write("balance", 2000)
print(f"Write took: {(time.time()-start)*1000:.1f}ms")
print(f"Primary reads: {store.eventual_read('balance', 0)}")
print(f"Replica reads: {store.eventual_read('balance', 2)}")  # Might be stale!
time.sleep(0.2)  # Wait for replication
print(f"Replica reads (after delay): {store.eventual_read('balance', 2)}")
```

### Java: Consistency Level Implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Demonstrates different consistency guarantees in a replicated store.
 */
public class ConsistencyDemo {
    
    enum ConsistencyLevel {
        STRONG,        // Wait for all replicas
        EVENTUAL,      // Write to one, replicate async
        READ_YOUR_OWN  // Track per-session writes
    }
    
    private final Map<String, VersionedValue>[] replicas;
    private final AtomicLong versionCounter = new AtomicLong(0);
    private final int numReplicas;
    private final ExecutorService replicator = Executors.newFixedThreadPool(4);
    
    // Per-session tracking for read-your-own-writes
    private final Map<String, Map<String, Long>> sessionVersions = new ConcurrentHashMap<>();

    record VersionedValue(Object value, long version, long timestamp) {}

    @SuppressWarnings("unchecked")
    public ConsistencyDemo(int numReplicas) {
        this.numReplicas = numReplicas;
        this.replicas = new ConcurrentHashMap[numReplicas];
        for (int i = 0; i < numReplicas; i++) {
            replicas[i] = new ConcurrentHashMap<>();
        }
    }

    public void write(String key, Object value, ConsistencyLevel level, String sessionId) {
        long version = versionCounter.incrementAndGet();
        VersionedValue vv = new VersionedValue(value, version, System.currentTimeMillis());
        
        switch (level) {
            case STRONG:
                // Synchronous: write to ALL replicas before returning
                for (var replica : replicas) {
                    replica.put(key, vv);
                }
                System.out.printf("[STRONG] Written key=%s, v=%d to ALL %d replicas%n",
                        key, version, numReplicas);
                break;
                
            case EVENTUAL:
                // Write to primary only; replicate async
                replicas[0].put(key, vv);
                replicator.submit(() -> {
                    for (int i = 1; i < numReplicas; i++) {
                        try { Thread.sleep(50); } catch (InterruptedException e) {}
                        replicas[i].put(key, vv);
                    }
                });
                System.out.printf("[EVENTUAL] Written key=%s, v=%d to primary only%n",
                        key, version);
                break;
                
            case READ_YOUR_OWN:
                // Write to primary + track session version
                replicas[0].put(key, vv);
                sessionVersions.computeIfAbsent(sessionId, k -> new ConcurrentHashMap<>())
                        .put(key, version);
                replicator.submit(() -> {
                    for (int i = 1; i < numReplicas; i++) {
                        replicas[i].put(key, vv);
                    }
                });
                break;
        }
    }

    public Optional<Object> read(String key, ConsistencyLevel level, String sessionId) {
        switch (level) {
            case STRONG:
                // Read from quorum (majority)
                List<VersionedValue> responses = new ArrayList<>();
                for (var replica : replicas) {
                    if (replica.containsKey(key)) {
                        responses.add(replica.get(key));
                    }
                }
                return responses.stream()
                        .max(Comparator.comparingLong(VersionedValue::version))
                        .map(VersionedValue::value);
                        
            case EVENTUAL:
                // Read from random replica (might be stale)
                int randomReplica = ThreadLocalRandom.current().nextInt(numReplicas);
                VersionedValue vv = replicas[randomReplica].get(key);
                return Optional.ofNullable(vv).map(VersionedValue::value);
                
            case READ_YOUR_OWN:
                // Find replica with at least our session's version
                long requiredVersion = sessionVersions
                        .getOrDefault(sessionId, Map.of())
                        .getOrDefault(key, 0L);
                
                for (var replica : replicas) {
                    VersionedValue val = replica.get(key);
                    if (val != null && val.version() >= requiredVersion) {
                        return Optional.of(val.value());
                    }
                }
                // Fallback to primary (guaranteed to have our writes)
                VersionedValue primary = replicas[0].get(key);
                return Optional.ofNullable(primary).map(VersionedValue::value);
                
            default:
                return Optional.empty();
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        ConsistencyDemo store = new ConsistencyDemo(3);
        
        // Strong: always see latest
        store.write("balance", 1000, ConsistencyLevel.STRONG, "session1");
        System.out.println("Strong read: " + store.read("balance", ConsistencyLevel.STRONG, "session1"));
        
        // Eventual: might see stale
        store.write("likes", 42, ConsistencyLevel.EVENTUAL, "session2");
        System.out.println("Eventual read (immediate): " + 
                store.read("likes", ConsistencyLevel.EVENTUAL, "session2"));
        Thread.sleep(200);
        System.out.println("Eventual read (after delay): " + 
                store.read("likes", ConsistencyLevel.EVENTUAL, "session2"));
    }
}
```

---

## Real-World Example

### Amazon DynamoDB: Eventual + Strong Options

```
DynamoDB per-request consistency choice:
═══════════════════════════════════════════════════

// Eventually Consistent Read (default)
aws dynamodb get-item \
  --table-name Users \
  --key '{"userId": {"S": "123"}}'

// Strongly Consistent Read (2x cost, higher latency)
aws dynamodb get-item \
  --table-name Users \
  --key '{"userId": {"S": "123"}}' \
  --consistent-read

When Amazon.com uses each:
─────────────────────────────────────────────────
Product page views → Eventually consistent (fast)
"Add to cart" → Eventually consistent (merge later)
Checkout/payment → Strongly consistent (must be accurate)
Order status → Read-your-own-writes (you see your order)
```

### Facebook/Meta: Causal Consistency for Social

```
Facebook's TAO (The Associations and Objects) cache:
═══════════════════════════════════════════════════

Problem: Billions of social graph queries per second.
         Strong consistency = too slow.
         Eventual consistency = weird behavior.

Solution: Causal consistency with "read-after-write"

Example:
1. Alice posts a photo (write to datacenter US-East)
2. Alice immediately sees her own photo (read-your-own-write)  
3. Bob in Europe sees the photo after ~1 second (eventual)
4. Bob comments on the photo
5. Everyone who sees Bob's comment ALSO sees Alice's photo
   (causal: comment depends on photo existing)

Architecture:
┌──────────┐     ┌──────────┐     ┌──────────┐
│ US-East  │────▶│ TAO Cache│────▶│  Europe  │
│ (Leader) │     │ (causal  │     │(Follower)│
│          │◀────│  tracking)│◀────│          │
└──────────┘     └──────────┘     └──────────┘

Causal metadata ensures: no user ever sees an effect 
                         without its cause.
```

### Google Spanner: External Consistency (Stronger than Linearizability!)

```
Spanner's TrueTime gives "external consistency":
═══════════════════════════════════════════════════

If transaction T1 commits before T2 starts (in real time),
then T1's timestamp < T2's timestamp. ALWAYS.

This is stronger than linearizability because it respects
REAL-WORLD time, not just program order.

How TrueTime works:
┌────────────────────────────────────────────────┐
│ TrueTime API returns: [earliest, latest]       │
│                                                │
│ tt.now() = {earliest: 10:00:00.003,           │
│             latest:   10:00:00.010}            │
│                                                │
│ Uncertainty window: 7ms                        │
│                                                │
│ To commit: wait until uncertainty passes       │
│ (called "commit-wait": ~7ms average)           │
│                                                │
│ Result: Timestamps are REAL, globally ordered  │
└────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Reality |
|---------|---------|
| "Eventual consistency means data can be lost" | No — data is preserved. It just takes time to propagate to all replicas. |
| "Strong consistency is always better" | It's SAFER but much SLOWER. Many use cases don't need it. |
| "Eventual means hours or days" | Usually milliseconds to seconds. "Eventually" in practice is very fast. |
| "We need strong consistency everywhere" | Only for operations where seeing stale data causes real harm (money, inventory). |
| "Causal consistency is 'basically' eventual" | No — causal provides ordering guarantees that eventual does not. It prevents reading effects before causes. |
| "Read-your-own-writes is easy" | In multi-region setups, this requires careful routing (sticky sessions or version tracking). |

---

## When to Use / When NOT to Use

### Strong Consistency:
- ✅ Bank balances, payment processing
- ✅ Inventory counts (prevent overselling)
- ✅ User authentication state
- ✅ Distributed locks, leader election
- ❌ Social media feeds, analytics, search

### Eventual Consistency:
- ✅ Social media timelines
- ✅ Product reviews and ratings
- ✅ Analytics/metrics counters
- ✅ DNS propagation
- ✅ CDN content delivery
- ❌ Financial transactions, inventory management

### Causal Consistency:
- ✅ Comment threads (reply must appear after parent)
- ✅ Collaborative editing (edits respect order)
- ✅ Social graphs (friend request before acceptance)
- ❌ When you need absolute latest value (use strong)

### Read-Your-Own-Writes:
- ✅ User profile updates (you see your changes)
- ✅ Form submissions (confirmation page shows your data)
- ✅ Shopping cart (items you just added appear)
- ❌ When other users must also see latest (use strong)

---

## Key Takeaways

- 🔑 **Consistency is a spectrum**, not binary. From linearizable (strongest) to eventual (weakest), with many useful levels in between.
- 🔑 **Strong consistency requires coordination** — nodes must communicate before responding, adding latency.
- 🔑 **Eventual consistency is fast but can show stale data** — acceptable when stale data doesn't cause harm.
- 🔑 **Causal consistency preserves cause-and-effect** — a powerful middle ground that prevents confusing user experiences.
- 🔑 **Read-your-own-writes** is the minimum consistency most user-facing apps need (users MUST see their own changes).
- 🔑 **Most real systems use MULTIPLE levels** — strong for payments, eventual for feeds, causal for social features.
- 🔑 **The "right" consistency depends on the business impact** of reading stale data, not technical preference.

---

## What's Next?

Now that you understand WHAT consistency guarantees are available, the next chapter explains HOW distributed systems achieve agreement among nodes: **[Distributed Consensus (Paxos, Raft)](./05-distributed-consensus.md)** — the algorithms that make strong consistency possible even when nodes crash.
