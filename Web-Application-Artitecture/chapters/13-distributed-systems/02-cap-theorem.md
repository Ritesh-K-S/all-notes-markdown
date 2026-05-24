# CAP Theorem — You Can Only Pick Two

> **What you'll learn**: The most famous theorem in distributed systems — why you must choose between Consistency, Availability, and Partition Tolerance, and how real databases make this impossible trade-off.

---

## Real-Life Analogy: Two Bank Branches

Imagine a bank with **two branches** (Branch A and Branch B) and you have $1,000 in your account.

**Normal day**: You withdraw $500 at Branch A. Branch A immediately calls Branch B: "Update the balance to $500." Both branches agree. Life is good.

**Now imagine the phone line between branches goes down** (a network partition):

You walk into Branch A and try to withdraw $800. The bank has three choices:

1. **Choose Consistency (refuse the withdrawal)**: "Sorry, we can't verify your balance with Branch B. Come back later." — You're unhappy, but no one loses money.

2. **Choose Availability (allow the withdrawal)**: "Sure, here's $800." — But meanwhile at Branch B, your spouse also withdraws $800. Now the bank has given out $1,600 from a $1,000 account!

3. **Ignore partitions** (pretend the line is fine): This isn't realistic — network failures WILL happen.

**This is the CAP Theorem in a nutshell.** When the network breaks, you pick Consistency OR Availability. You cannot have both.

---

## Core Concept Explained Step-by-Step

### The Three Properties

**CAP Theorem** (proved by Eric Brewer in 2000, formally proven by Seth Gilbert & Nancy Lynch in 2002):

In a distributed data store, you can guarantee AT MOST two of these three properties simultaneously:

```
                    ╔═══════════════╗
                    ║  CONSISTENCY  ║
                    ║  (C)          ║
                    ╚═══════╤═══════╝
                           ╱ ╲
                          ╱   ╲
                         ╱     ╲
                        ╱  PICK  ╲
                       ╱   TWO    ╲
                      ╱             ╲
        ╔════════════╧═══╗    ╔═════╧════════════╗
        ║  AVAILABILITY  ║    ║    PARTITION     ║
        ║  (A)           ║    ║   TOLERANCE (P)  ║
        ╚════════════════╝    ╚══════════════════╝
```

### What Each Property Means (Precisely)

| Property | Definition | Plain English |
|----------|-----------|---------------|
| **Consistency (C)** | Every read receives the most recent write or an error | All nodes see the SAME data at the SAME time |
| **Availability (A)** | Every request receives a non-error response (no guarantee it's the latest) | Every request gets SOME answer (system never says "unavailable") |
| **Partition Tolerance (P)** | The system continues to operate despite network partitions between nodes | System works even when some nodes can't talk to each other |

### Why You Must Choose

In any real distributed system, **network partitions WILL happen**. Cables get cut, switches fail, data centers lose connectivity. So **P is not optional** — you MUST have Partition Tolerance.

This means the real choice is:

```
  Network Partition Happens! 🔥
  ────────────────────────────────────────
  
  ┌──────────┐         ✂️ (cut!)         ┌──────────┐
  │  Node A  │ ─── ─── ─── ─── ───X──── │  Node B  │
  └──────────┘                            └──────────┘
  
  Now what?
  
  Option 1: CP (Consistency + Partition Tolerance)
  ─────────────────────────────────────────────────
  → Stop serving requests until partition heals
  → "Sorry, system unavailable" (sacrifices A)
  → But data is ALWAYS correct
  
  Option 2: AP (Availability + Partition Tolerance)
  ─────────────────────────────────────────────────
  → Keep serving requests on both sides
  → Each side may have STALE/DIFFERENT data (sacrifices C)
  → But system NEVER goes down
```

### The Three Combinations

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CAP COMBINATIONS                            │
├──────────┬────────────────────────────────────────────┬─────────────┤
│   Type   │  Description                               │  Examples   │
├──────────┼────────────────────────────────────────────┼─────────────┤
│          │  Strong consistency + partition tolerance   │             │
│   CP     │  System may reject requests during         │  MongoDB*   │
│          │  partition to maintain consistency          │  HBase      │
│          │                                            │  Redis**    │
│          │                                            │  ZooKeeper  │
├──────────┼────────────────────────────────────────────┼─────────────┤
│          │  High availability + partition tolerance    │             │
│   AP     │  System always responds but may return     │  Cassandra  │
│          │  stale data during partition               │  DynamoDB   │
│          │                                            │  CouchDB    │
│          │                                            │  Riak       │
├──────────┼────────────────────────────────────────────┼─────────────┤
│          │  Consistency + availability (no partition)  │             │
│   CA     │  Only possible on a SINGLE machine or      │  PostgreSQL │
│          │  network that NEVER fails (theoretical)    │  (single)   │
│          │                                            │  MySQL      │
│          │                                            │  (single)   │
└──────────┴────────────────────────────────────────────┴─────────────┘

* MongoDB with majority writes is CP; with w:1 it behaves more AP
** Redis Cluster can lose writes during failover (not strictly CP)
```

---

## How It Works Internally

### Scenario: CP System During Partition

```
BEFORE PARTITION:
                 ┌─────────────┐
 Write "x=5" ──▶│   Node A    │──── replicates ────▶┌─────────────┐
                 │  (Primary)  │                     │   Node B    │
                 │   x = 5     │◀── acknowledges ───│   x = 5     │
                 └─────────────┘                     └─────────────┘

Both nodes have x=5. Reads from EITHER node return 5. ✅

DURING PARTITION:
                 ┌─────────────┐        ✂️          ┌─────────────┐
 Write "x=7" ──▶│   Node A    │──── CANNOT ────X──▶│   Node B    │
                 │  (Primary)  │     reach!          │   x = 5     │
                 │   x = ???   │                     │  (stale!)   │
                 └─────────────┘                     └─────────────┘

CP System Response:
─────────────────────────────────────
Node A: "I cannot confirm Node B received the update."
        "REJECTING the write. Error returned to client."
        
Result: Write FAILS. But data is CONSISTENT everywhere.
        Client must retry later.
```

### Scenario: AP System During Partition

```
DURING PARTITION:
                 ┌─────────────┐        ✂️          ┌─────────────┐
 Write "x=7" ──▶│   Node A    │──── CANNOT ────X──▶│   Node B    │
                 │             │     reach!          │             │
                 │   x = 7 ✅  │                     │   x = 5     │
                 └─────────────┘                     └─────────────┘

AP System Response:
─────────────────────────────────────
Node A: "Write accepted! x = 7" (locally)
Node B: (still serving reads with x = 5 — STALE!)

A client reading from Node A sees: x = 7
A client reading from Node B sees: x = 5  ← INCONSISTENCY!

AFTER PARTITION HEALS:
─────────────────────────────────────
Nodes sync up: "Hey, I have x=7, you have x=5... 
               Which one wins?"
               
→ Conflict resolution needed! (Last-write-wins, vector clocks, etc.)
```

### Quorum-Based Systems: Tunable Consistency

Many modern databases let you TUNE the trade-off per operation:

```
N = Total replicas = 3
W = Write quorum (must acknowledge before success)
R = Read quorum (must respond before returning)

RULE: If W + R > N → Strong Consistency (reads always see latest writes)

┌─────────────────────────────────────────────────────────────────┐
│  Example with N=3:                                              │
│                                                                 │
│  W=3, R=1 → CP (all nodes must confirm write; any node can read)│
│  W=1, R=3 → CP (fast writes; read from all to get latest)      │
│  W=2, R=2 → CP (balanced: majority write + majority read)      │
│  W=1, R=1 → AP (fast but may read stale data!)                 │
└─────────────────────────────────────────────────────────────────┘

Visual: Write with W=2, N=3
─────────────────────────────────────
Client ──write──▶ Node 1 ✅ ACK
                  Node 2 ✅ ACK  ← 2 ACKs received = success!
                  Node 3 ❌ (slow/partitioned — doesn't matter)
                  
Response to client: "Write successful!"
```

---

## Code Examples

### Python: Demonstrating CAP Trade-offs

```python
import time
import random
from enum import Enum
from typing import Optional, Tuple

class ConsistencyMode(Enum):
    CP = "cp"  # Reject writes during partition
    AP = "ap"  # Accept writes, allow inconsistency

class DistributedStore:
    """Simulates a distributed key-value store with CAP trade-offs."""
    
    def __init__(self, mode: ConsistencyMode, nodes: int = 3):
        self.mode = mode
        self.nodes = [{} for _ in range(nodes)]  # Each node has its own data
        self.partitioned = False  # Simulates network partition
    
    def write(self, key: str, value: str) -> Tuple[bool, str]:
        """Write to the distributed store."""
        # Write to primary (node 0)
        if self.mode == ConsistencyMode.CP:
            return self._cp_write(key, value)
        else:
            return self._ap_write(key, value)
    
    def _cp_write(self, key: str, value: str) -> Tuple[bool, str]:
        """CP: Reject write if we can't reach majority."""
        reachable = self._count_reachable_nodes()
        majority = len(self.nodes) // 2 + 1
        
        if reachable < majority:
            # Cannot guarantee consistency — REJECT
            return False, f"ERROR: Only {reachable}/{len(self.nodes)} nodes reachable. Need {majority}."
        
        # Write to all reachable nodes
        for i, node in enumerate(self.nodes):
            if self._can_reach(i):
                node[key] = value
        
        return True, f"OK: Written to {reachable} nodes (consistent)."
    
    def _ap_write(self, key: str, value: str) -> Tuple[bool, str]:
        """AP: Always accept write, replicate best-effort."""
        # Always write locally
        self.nodes[0][key] = value
        replicated_to = 1
        
        # Try to replicate to others
        for i in range(1, len(self.nodes)):
            if self._can_reach(i):
                self.nodes[i][key] = value
                replicated_to += 1
        
        return True, f"OK: Written (replicated to {replicated_to}/{len(self.nodes)} nodes)."
    
    def read(self, key: str, from_node: int = 0) -> Optional[str]:
        """Read from a specific node."""
        return self.nodes[from_node].get(key, None)
    
    def _can_reach(self, node_idx: int) -> bool:
        """Simulate network partition."""
        if not self.partitioned:
            return True
        # During partition, only node 0 is reachable
        return node_idx == 0
    
    def _count_reachable_nodes(self) -> int:
        return sum(1 for i in range(len(self.nodes)) if self._can_reach(i))
    
    def simulate_partition(self):
        """Simulate a network partition."""
        self.partitioned = True
        print("🔥 NETWORK PARTITION! Nodes cannot communicate.")
    
    def heal_partition(self):
        """Simulate partition recovery."""
        self.partitioned = False
        print("✅ Network healed. Nodes can communicate again.")


# --- Demo ---
print("=== CP Mode (like ZooKeeper) ===")
cp_store = DistributedStore(ConsistencyMode.CP)
print(cp_store.write("user:1", "Alice"))  # OK
cp_store.simulate_partition()
print(cp_store.write("user:2", "Bob"))    # FAILS — can't reach majority

print("\n=== AP Mode (like Cassandra) ===")
ap_store = DistributedStore(ConsistencyMode.AP)
print(ap_store.write("user:1", "Alice"))  # OK
ap_store.simulate_partition()
print(ap_store.write("user:2", "Bob"))    # OK — but only on node 0!
# Reading from different nodes gives different results:
print(f"Node 0 sees: {ap_store.read('user:2', 0)}")  # "Bob"
print(f"Node 1 sees: {ap_store.read('user:2', 1)}")  # None (stale!)
```

### Java: Quorum-Based Read/Write

```java
import java.util.*;
import java.util.concurrent.*;

/**
 * Demonstrates quorum-based consistency in a distributed store.
 * W + R > N guarantees strong consistency.
 */
public class QuorumStore {
    private final int totalNodes;      // N
    private final int writeQuorum;     // W
    private final int readQuorum;      // R
    private final Map<String, String>[] nodes;
    private final Set<Integer> partitionedNodes = new HashSet<>();

    @SuppressWarnings("unchecked")
    public QuorumStore(int n, int w, int r) {
        this.totalNodes = n;
        this.writeQuorum = w;
        this.readQuorum = r;
        this.nodes = new ConcurrentHashMap[n];
        for (int i = 0; i < n; i++) {
            nodes[i] = new ConcurrentHashMap<>();
        }
        System.out.printf("Store: N=%d, W=%d, R=%d | Strong consistency: %s%n",
                n, w, r, (w + r > n) ? "YES" : "NO");
    }

    public boolean write(String key, String value) {
        int acks = 0;
        for (int i = 0; i < totalNodes; i++) {
            if (!partitionedNodes.contains(i)) {
                nodes[i].put(key, value);
                acks++;
            }
        }
        
        if (acks >= writeQuorum) {
            System.out.printf("WRITE OK: key=%s, value=%s (acks=%d/%d)%n",
                    key, value, acks, writeQuorum);
            return true;
        } else {
            System.out.printf("WRITE FAILED: key=%s (acks=%d, need=%d)%n",
                    key, acks, writeQuorum);
            return false;
        }
    }

    public Optional<String> read(String key) {
        List<String> responses = new ArrayList<>();
        for (int i = 0; i < totalNodes; i++) {
            if (!partitionedNodes.contains(i)) {
                String val = nodes[i].get(key);
                if (val != null) responses.add(val);
            }
        }
        
        if (responses.size() >= readQuorum) {
            // Return the "latest" value (in real systems, use version numbers)
            String result = responses.get(responses.size() - 1);
            System.out.printf("READ OK: key=%s → %s (responses=%d/%d)%n",
                    key, result, responses.size(), readQuorum);
            return Optional.of(result);
        } else {
            System.out.printf("READ FAILED: key=%s (responses=%d, need=%d)%n",
                    key, responses.size(), readQuorum);
            return Optional.empty();
        }
    }

    public void partitionNode(int nodeId) {
        partitionedNodes.add(nodeId);
        System.out.printf("🔥 Node %d partitioned!%n", nodeId);
    }

    public static void main(String[] args) {
        // Strong consistency: W=2, R=2, N=3 (W+R=4 > N=3)
        QuorumStore store = new QuorumStore(3, 2, 2);
        
        store.write("balance", "1000");
        store.read("balance");
        
        // Partition one node
        store.partitionNode(2);
        store.write("balance", "500");   // Still works (2 nodes left ≥ W=2)
        store.read("balance");           // Still works (2 nodes left ≥ R=2)
        
        // Partition another node
        store.partitionNode(1);
        store.write("balance", "200");   // FAILS (1 node < W=2)
        store.read("balance");           // FAILS (1 node < R=2)
    }
}
```

---

## Real-World Example

### DynamoDB: AP with Tunable Consistency

Amazon's DynamoDB defaults to **Eventually Consistent Reads** (AP) but offers **Strongly Consistent Reads** (CP) per-request:

```
DynamoDB Write Flow:
═══════════════════════════════════════════════

Client ──write──▶ DynamoDB
                      │
                      ├──▶ Replica 1 (primary) ✅
                      ├──▶ Replica 2            ✅  (W=2 of 3)
                      └──▶ Replica 3            ⏳  (async)
                      
Response: "Write successful!" (after 2 acks)

Eventually Consistent Read (default, faster):
─────────────────────────────────────────
→ Reads from ANY replica (might get stale data from Replica 3)
→ 50% cheaper, lower latency

Strongly Consistent Read (opt-in, slower):
─────────────────────────────────────────
→ Reads from the primary replica only
→ Always returns latest data
→ Higher latency, more expensive
```

### Google Spanner: Effectively "CA" with Hardware Help

Google Spanner achieves something remarkable — it's essentially CP but with such high availability that it approaches CA:

```
How Spanner "breaks" CAP:
═══════════════════════════════════════════════

Problem: CAP says you can't have C + A during partition
Spanner's answer: "What if partitions almost NEVER happen?"

Techniques:
1. Private global fiber network (not public internet)
2. TrueTime API (GPS + atomic clocks in every data center)
3. Redundant network paths everywhere
4. Partitions are so rare they're treated as "disasters"

Result: Strong consistency + 99.999% availability
        (technically CP, but partition is ultra-rare)
```

---

## Common Mistakes / Pitfalls

| Mistake | Reality |
|---------|---------|
| "I'll build a CA system" | Impossible in a real distributed system. Network partitions WILL happen. |
| "CAP means I always lose one property" | No! You only lose one DURING a partition. In normal operation, you can have all three. |
| "MongoDB is CP, Cassandra is AP — that's fixed" | Many systems are TUNABLE. You can configure consistency levels per operation. |
| "Consistency in CAP = ACID Consistency" | Different concepts! CAP Consistency = linearizability. ACID Consistency = data integrity rules. |
| "AP means data loss" | No — AP means TEMPORARY inconsistency. Data syncs up after partition heals (eventual consistency). |
| "I should always choose CP for safety" | Not necessarily. For a social media feed, AP is fine. For a bank balance, CP is critical. |

---

## When to Use / When NOT to Use

### Choose CP When:
- ✅ Financial transactions (bank accounts, payments)
- ✅ Inventory management (don't oversell!)
- ✅ Configuration/coordination (ZooKeeper, etcd)
- ✅ User authentication (login state must be accurate)
- ✅ Leader election and distributed locks

### Choose AP When:
- ✅ Social media feeds (showing a slightly stale post is OK)
- ✅ Shopping carts (merge conflicts later)
- ✅ DNS (cached records are acceptable)
- ✅ Content delivery (CDN can serve slightly old content)
- ✅ Analytics/metrics (approximate counts are fine)
- ✅ Search indexes (slight delay in indexing is acceptable)

### Decision Framework:

```
"What happens if a user sees stale data for 5 seconds?"

If answer is "people lose money" → CP
If answer is "minor inconvenience" → AP
If answer is "nobody notices" → AP (definitely)
```

---

## Key Takeaways

- 🔑 **CAP Theorem**: In a distributed system, when a network partition occurs, you must choose between Consistency and Availability.
- 🔑 **Partition Tolerance is mandatory** — you don't really choose between C, A, and P. You choose between CP and AP.
- 🔑 **The trade-off only matters during partitions** — in normal operation, you can have all three.
- 🔑 **Most modern databases are tunable** — you choose the trade-off per operation (DynamoDB, Cassandra, MongoDB).
- 🔑 **CAP Consistency ≠ ACID Consistency** — don't confuse them. CAP is about linearizability across nodes.
- 🔑 **Your choice depends on your use case** — financial data needs CP; social feeds are fine with AP.
- 🔑 CAP is a simplified model — real systems have more nuances (see PACELC in the next chapter).

---

## What's Next?

CAP is a simplified model that only considers the partition scenario. But what about the trade-offs when there's NO partition? That's exactly what the **[PACELC Theorem — Beyond CAP](./03-pacelc-theorem.md)** addresses. It extends CAP by asking: "Even when things are running normally, do you prefer lower latency or stronger consistency?"
