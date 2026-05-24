# PACELC Theorem — Beyond CAP

> **What you'll learn**: Why CAP Theorem is incomplete, what trade-offs exist even when there's NO partition, and how PACELC gives a more realistic framework for understanding distributed system behavior.

---

## Real-Life Analogy: The Airport Security Trade-off

Think of airport security:

**During a bomb threat (partition/crisis):**
- You either **lock down the airport** (choose Consistency/safety — no one flies) 
- Or **keep flights running** (choose Availability — risk something slipping through)

That's CAP. But what about **normal days with no threats?**

Even on a normal day, you still face a trade-off:
- **More security checks** = safer but **slower** (everyone waits in long lines)
- **Fewer checks** = faster but **less thorough** (people board quickly)

**PACELC says**: Even when everything is working fine (no partition), you still must choose between **Latency** and **Consistency**. There's ALWAYS a trade-off, not just during failures.

---

## Core Concept Explained Step-by-Step

### Why CAP Isn't Enough

CAP only tells you what happens **during a partition**. But partitions are rare — most of the time, your system runs normally. CAP says nothing about the trade-offs you make during normal operation.

```
CAP's blind spot:
═══════════════════════════════════════════════════

Timeline of a system's life:
───────────────────────────────────────────────────
|  Normal  |  Normal  | PARTITION |  Normal  |  Normal  |
|  99.9%   |  of the  |  0.1% of  |  of the  |  99.9%  |
|  of time |  time    |   time    |  time    |  of time |
───────────────────────────────────────────────────
     ↑          ↑          ↑          ↑          ↑
     ?          ?       CAP tells     ?          ?
                       you about
                         this

What about the 99.9% of normal operation time?
That's what PACELC addresses!
```

### The PACELC Statement

**PACELC** (pronounced "pass-elk") was proposed by Daniel Abadi in 2012:

```
╔═════════════════════════════════════════════════════════════════════╗
║                          PACELC                                    ║
║                                                                    ║
║  IF there is a Partition (P):                                     ║
║      → Choose between Availability (A) and Consistency (C)        ║
║                                                                    ║
║  ELSE (E) — when system is running normally:                      ║
║      → Choose between Latency (L) and Consistency (C)             ║
║                                                                    ║
╚═════════════════════════════════════════════════════════════════════╝

Read as: "If Partition → A or C; Else → L or C"
         P-A-C-E-L-C
```

### Breaking It Down

```
┌─────────────────────────────────────────────────────────────────┐
│                    PACELC Decision Tree                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                   Is there a network partition?                  │
│                         /              \                         │
│                       YES               NO                      │
│                      /                    \                      │
│              [PAC Part]              [ELC Part]                  │
│               /      \                /       \                 │
│          Choose A   Choose C    Choose L    Choose C            │
│          (Available)(Consistent)(Low Latency)(Consistent)       │
│                                                                 │
│   Examples:         Examples:                                   │
│   PA: Cassandra    EL: Cassandra (fast reads, eventual)         │
│   PC: MongoDB      EC: MongoDB (waits for consistency)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The "Else Latency vs Consistency" Trade-off

Why does normal operation involve a trade-off? Because **replication takes time**:

```
SCENARIO: Write "x=42" to a 3-node cluster

Option A: Low Latency (EL)
──────────────────────────────────────────────
Client ──write──▶ Node 1 ✅ (primary)
                      │
                      ├── async replicate ──▶ Node 2 (happens later)
                      └── async replicate ──▶ Node 3 (happens later)
                      
Response time: ~2ms (just primary write)
Risk: If primary crashes before replication, data LOST

Option B: Strong Consistency (EC)
──────────────────────────────────────────────
Client ──write──▶ Node 1 ✅ (primary)
                      │
                      ├── sync replicate ──▶ Node 2 ✅ (wait for ACK)
                      └── sync replicate ──▶ Node 3 ✅ (wait for ACK)
                      
Response time: ~15ms (waited for all replicas)
Guarantee: Data is safe on 3 nodes before confirming

The EXTRA 13ms is the "consistency tax" you pay.
At scale, this adds up enormously.
```

---

## How It Works Internally

### PACELC Classification of Real Systems

```
╔══════════════════╦════════════════════╦════════════════════╗
║     System       ║  During Partition  ║  Normal Operation  ║
║                  ║     (PAC)          ║      (ELC)         ║
╠══════════════════╬════════════════════╬════════════════════╣
║ DynamoDB         ║   PA (available)   ║   EL (low latency) ║
║ Cassandra        ║   PA (available)   ║   EL (low latency) ║
║ Riak             ║   PA (available)   ║   EL (low latency) ║
╠══════════════════╬════════════════════╬════════════════════╣
║ MongoDB (default)║   PC (consistent)  ║   EC (consistent)  ║
║ HBase            ║   PC (consistent)  ║   EC (consistent)  ║
║ ZooKeeper        ║   PC (consistent)  ║   EC (consistent)  ║
╠══════════════════╬════════════════════╬════════════════════╣
║ Cosmos DB        ║   PA (available)   ║   EL (low latency) ║
║   (tunable!)     ║   or PC            ║   or EC            ║
╠══════════════════╬════════════════════╬════════════════════╣
║ Google Spanner   ║   PC (consistent)  ║   EC (consistent)  ║
║                  ║                    ║   but low latency!* ║
╚══════════════════╩════════════════════╩════════════════════╝

* Spanner uses TrueTime (atomic clocks) to minimize the latency cost of consistency
```

### Why Latency vs Consistency is Unavoidable

The speed of light creates a **physical minimum** for replication:

```
Distance between data centers:
═══════════════════════════════════════════════

New York ←──── 3,944 miles ────→ London
Light speed: ~186,000 miles/sec
Minimum RTT: ~42ms (just physics!)

For strong consistency across NY and London:
─ Write must reach BOTH before confirming
─ Minimum latency: ~42ms (even with perfect network)
─ Actual latency: ~60-80ms (with processing overhead)

For eventual consistency (write locally):
─ Write confirms immediately: ~2ms
─ Replication happens in background

The difference: 2ms vs 60ms
For 1000 requests/sec: 2 seconds vs 60 seconds of waiting
```

### Tunable Consistency in Practice

Modern databases let you pick your PACELC position per operation:

```
CASSANDRA CONSISTENCY LEVELS:
═══════════════════════════════════════════════

Write Consistency:
┌───────────────┬──────────────────────────────────────────┐
│    Level      │  Behavior                                │
├───────────────┼──────────────────────────────────────────┤
│ ONE           │ Write to 1 node → respond (EL: fast!)    │
│ QUORUM        │ Write to majority → respond (middle)     │
│ ALL           │ Write to ALL nodes → respond (EC: slow!) │
│ LOCAL_QUORUM  │ Majority in local DC only (geo-aware)    │
└───────────────┴──────────────────────────────────────────┘

Read Consistency:
┌───────────────┬──────────────────────────────────────────┐
│    Level      │  Behavior                                │
├───────────────┼──────────────────────────────────────────┤
│ ONE           │ Read from closest node (fast, may stale) │
│ QUORUM        │ Read from majority (consistent, slower)  │
│ ALL           │ Read from ALL (slowest, most consistent) │
└───────────────┴──────────────────────────────────────────┘

You can mix! E.g., Write with QUORUM + Read with QUORUM = Strong Consistency
But Write with ONE + Read with ONE = Fast but potentially stale
```

---

## Code Examples

### Python: PACELC-Aware Distributed Store

```python
import time
import random
from enum import Enum
from dataclasses import dataclass
from typing import Optional, List

class PACELCMode(Enum):
    PA_EL = "pa_el"  # Partition→Available, Else→Low Latency (Cassandra-like)
    PC_EC = "pc_ec"  # Partition→Consistent, Else→Consistent (MongoDB-like)
    PA_EC = "pa_ec"  # Partition→Available, Else→Consistent (rare hybrid)
    PC_EL = "pc_el"  # Partition→Consistent, Else→Low Latency (rare hybrid)

@dataclass
class WriteResult:
    success: bool
    latency_ms: float
    replicas_confirmed: int
    message: str

class PACELCStore:
    """Demonstrates PACELC trade-offs in a distributed store."""
    
    def __init__(self, mode: PACELCMode, num_nodes: int = 3, 
                 replication_delay_ms: float = 15.0):
        self.mode = mode
        self.num_nodes = num_nodes
        self.replication_delay = replication_delay_ms
        self.nodes = [{} for _ in range(num_nodes)]
        self.partitioned = False
    
    def write(self, key: str, value: str) -> WriteResult:
        """Write with PACELC-aware behavior."""
        start = time.time()
        
        if self.partitioned:
            return self._write_during_partition(key, value, start)
        else:
            return self._write_normal(key, value, start)
    
    def _write_during_partition(self, key, value, start) -> WriteResult:
        """PAC part: Choose Availability or Consistency during partition."""
        if self.mode in (PACELCMode.PA_EL, PACELCMode.PA_EC):
            # PA: Accept write on available node (sacrifice consistency)
            self.nodes[0][key] = value
            latency = (time.time() - start) * 1000
            return WriteResult(True, latency, 1,
                "PA: Write accepted on 1 node (others unreachable, may be inconsistent)")
        else:
            # PC: Reject write (sacrifice availability)
            latency = (time.time() - start) * 1000
            return WriteResult(False, latency, 0,
                "PC: Write REJECTED — cannot ensure consistency during partition")
    
    def _write_normal(self, key, value, start) -> WriteResult:
        """ELC part: Choose Latency or Consistency during normal operation."""
        if self.mode in (PACELCMode.PA_EL, PACELCMode.PC_EL):
            # EL: Write locally, replicate async (fast but eventually consistent)
            self.nodes[0][key] = value
            latency = (time.time() - start) * 1000
            
            # Simulate async replication (happens "later")
            # In real system, this would be background thread
            for i in range(1, self.num_nodes):
                self.nodes[i][key] = value  # Pretend this is async
            
            return WriteResult(True, latency, 1,
                f"EL: Fast write ({latency:.1f}ms), async replication")
        else:
            # EC: Wait for all replicas (slow but strongly consistent)
            self.nodes[0][key] = value
            
            # Simulate synchronous replication delay
            time.sleep(self.replication_delay / 1000)
            for i in range(1, self.num_nodes):
                self.nodes[i][key] = value
            
            latency = (time.time() - start) * 1000
            return WriteResult(True, latency, self.num_nodes,
                f"EC: Consistent write ({latency:.1f}ms), all replicas confirmed")


# --- Demo: Compare PA/EL vs PC/EC ---
print("=" * 60)
print("PACELC Demonstration")
print("=" * 60)

# Cassandra-like: PA/EL
cassandra = PACELCStore(PACELCMode.PA_EL, replication_delay_ms=15)
print("\n--- Cassandra-like (PA/EL) ---")
print("Normal:", cassandra.write("key1", "value1"))
cassandra.partitioned = True
print("Partition:", cassandra.write("key2", "value2"))

# MongoDB-like: PC/EC
mongo = PACELCStore(PACELCMode.PC_EC, replication_delay_ms=15)
print("\n--- MongoDB-like (PC/EC) ---")
print("Normal:", mongo.write("key1", "value1"))
mongo.partitioned = True
print("Partition:", mongo.write("key2", "value2"))
```

### Java: Latency vs Consistency Measurement

```java
import java.util.*;
import java.util.concurrent.*;

/**
 * Demonstrates the Latency vs Consistency trade-off (ELC part of PACELC).
 * Shows measurable latency difference between EL and EC approaches.
 */
public class PACELCDemo {
    
    enum Mode { EL, EC }
    
    private final Mode mode;
    private final int numReplicas;
    private final long replicationDelayMs;
    private final Map<String, String>[] replicas;

    @SuppressWarnings("unchecked")
    public PACELCDemo(Mode mode, int numReplicas, long replicationDelayMs) {
        this.mode = mode;
        this.numReplicas = numReplicas;
        this.replicationDelayMs = replicationDelayMs;
        this.replicas = new ConcurrentHashMap[numReplicas];
        for (int i = 0; i < numReplicas; i++) {
            replicas[i] = new ConcurrentHashMap<>();
        }
    }

    public long write(String key, String value) {
        long start = System.nanoTime();
        
        // Always write to primary
        replicas[0].put(key, value);
        
        if (mode == Mode.EC) {
            // EC: Synchronous replication — wait for ALL replicas
            for (int i = 1; i < numReplicas; i++) {
                simulateNetworkDelay();
                replicas[i].put(key, value);
            }
        } else {
            // EL: Async replication — fire and forget
            CompletableFuture.runAsync(() -> {
                for (int i = 1; i < numReplicas; i++) {
                    simulateNetworkDelay();
                    replicas[i].put(key, value);
                }
            });
        }
        
        return (System.nanoTime() - start) / 1_000_000; // Return latency in ms
    }

    private void simulateNetworkDelay() {
        try { Thread.sleep(replicationDelayMs); } 
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    public static void main(String[] args) {
        int operations = 100;
        
        // EL mode: Low latency, eventual consistency
        PACELCDemo elStore = new PACELCDemo(Mode.EL, 3, 10);
        long elTotal = 0;
        for (int i = 0; i < operations; i++) {
            elTotal += elStore.write("key" + i, "value" + i);
        }
        
        // EC mode: Higher latency, strong consistency
        PACELCDemo ecStore = new PACELCDemo(Mode.EC, 3, 10);
        long ecTotal = 0;
        for (int i = 0; i < operations; i++) {
            ecTotal += ecStore.write("key" + i, "value" + i);
        }
        
        System.out.println("═══════════════════════════════════════");
        System.out.println("PACELC Latency Comparison (" + operations + " writes)");
        System.out.println("═══════════════════════════════════════");
        System.out.printf("EL (low latency):     avg %dms/write%n", elTotal / operations);
        System.out.printf("EC (consistent):      avg %dms/write%n", ecTotal / operations);
        System.out.printf("Consistency tax:      ~%dx slower%n", 
                ecTotal / Math.max(1, elTotal));
    }
}
```

---

## Real-World Example

### Amazon DynamoDB: PA/EL

```
DynamoDB's PACELC position: PA/EL
═══════════════════════════════════════════════

DURING PARTITION (PA):
─────────────────────────────────────────
- DynamoDB keeps accepting writes
- Writes go to available nodes
- Conflicts resolved later with "last writer wins" or custom logic
- Shopping carts MUST stay available (lost sales > stale data)

NORMAL OPERATION (EL):
─────────────────────────────────────────
- Default: eventually consistent reads (~few ms latency)
- Optional: strongly consistent reads (~2x latency, costs more)
- Replication happens async to secondary regions

Why this makes sense for Amazon:
- "Add to cart" should NEVER fail (lost revenue!)
- If two users add same last item, resolve during checkout
- Low latency = better user experience = more sales
```

### Google Spanner: PC/EC (but fast!)

```
Spanner's PACELC position: PC/EC
═══════════════════════════════════════════════

DURING PARTITION (PC):
─────────────────────────────────────────
- Spanner refuses to serve if it can't guarantee consistency
- This is acceptable because Google's private network makes
  partitions EXTREMELY rare (< 0.001% of time)

NORMAL OPERATION (EC):
─────────────────────────────────────────
- Strong consistency: every read sees latest write
- TrueTime API minimizes the "consistency tax":
  
  Traditional EC: wait for all replicas (50-100ms across globe)
  Spanner EC: wait for TrueTime uncertainty (~7ms average!)
  
  How? GPS + atomic clocks give tight time bounds.
  Instead of "wait for all nodes," it's "wait until
  we're SURE no conflicting write could exist."

The secret sauce:
┌────────────────────────────────────────────────┐
│  Every Google data center has:                 │
│  - GPS receivers (satellite time)             │
│  - Atomic clocks (local reference)            │
│  - TrueTime daemon combining both             │
│                                                │
│  Result: Clock uncertainty < 7ms              │
│  → Wait only 7ms for consistency guarantee!   │
└────────────────────────────────────────────────┘
```

### Cosmos DB: Fully Tunable

Azure Cosmos DB lets you pick ANY position on the PACELC spectrum:

```
Cosmos DB Consistency Levels:
═══════════════════════════════════════════════

STRONG          ──── PC/EC (slowest, always consistent)
BOUNDED STALENESS ── PC/EC (allow K versions or T time stale)
SESSION         ──── PA/EC (your own writes consistent)
CONSISTENT PREFIX── PA/EL (order preserved, may be stale)
EVENTUAL        ──── PA/EL (fastest, may read any version)

                    ◄─── More Consistent ── More Available ───►
                    ◄─── Higher Latency ─── Lower Latency ────►
                    ◄─── Lower Throughput ── Higher Throughput ►

Trade-off is explicit and per-account/per-request configurable!
```

---

## Common Mistakes / Pitfalls

| Mistake | Reality |
|---------|---------|
| "PACELC replaces CAP" | No — PACELC **extends** CAP. CAP is still valid; PACELC adds the normal-operation dimension. |
| "EL means data loss" | No — data is still replicated, just asynchronously. Loss only occurs if primary fails before replication completes. |
| "EC is always better because consistency is important" | EC adds latency to EVERY operation. At 10,000 req/sec, even 10ms extra = 100 seconds of waiting per second. |
| "I should pick one mode for my entire system" | Different operations have different needs. User registration = EC; activity feed = EL. |
| "Low latency = bad consistency" | Not always — Spanner achieves both via hardware (TrueTime). But it costs millions in infrastructure. |

---

## When to Use / When NOT to Use

### Choose PA/EL (Cassandra, DynamoDB) When:
- ✅ User-facing operations where latency directly impacts revenue
- ✅ Shopping carts, social feeds, real-time analytics
- ✅ Multi-region deployments where cross-region latency is high
- ✅ Write-heavy workloads where sync replication is a bottleneck

### Choose PC/EC (Spanner, ZooKeeper) When:
- ✅ Financial transactions that CANNOT be wrong
- ✅ Coordination services (locks, leader election, config)
- ✅ Inventory systems where overselling is unacceptable
- ✅ Regulatory requirements demand consistent views

### Choose Tunable (Cosmos DB, Cassandra) When:
- ✅ Different operations need different guarantees
- ✅ You want to optimize per use case within one system
- ✅ You're willing to manage the complexity of multiple consistency levels

---

## Key Takeaways

- 🔑 **CAP only covers partition scenarios** — PACELC extends it to cover normal operation too.
- 🔑 **PACELC = "If Partition → A or C; Else → L or C"** — there's ALWAYS a trade-off.
- 🔑 **The "Else" trade-off matters MORE** because systems operate normally 99.9%+ of the time.
- 🔑 **EL (low latency)** = writes confirmed before full replication; great for user-facing speed.
- 🔑 **EC (consistent)** = writes wait for replication; guarantees everyone sees the same data.
- 🔑 **Modern databases are tunable** — Cosmos DB, Cassandra, and DynamoDB let you choose per operation.
- 🔑 **Google Spanner minimizes the EC latency cost** using TrueTime (GPS + atomic clocks) — but this requires enormous infrastructure investment.

---

## What's Next?

Now that you understand the high-level trade-offs (CAP and PACELC), let's dive deep into what "Consistency" actually means at a technical level. The next chapter covers **[Consistency Models (Strong, Eventual, Causal)](./04-consistency-models.md)** — the different guarantees a system can provide about data freshness and ordering.
