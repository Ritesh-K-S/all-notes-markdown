# Vector Clocks & Conflict Resolution

> **What you'll learn**: How distributed systems track the order of events without a global clock, detect conflicts between concurrent updates, and resolve those conflicts — using Lamport timestamps and vector clocks.

---

## Real-Life Analogy: Collaborative Document Editing

Imagine Alice and Bob are editing a shared document, but offline (no internet):

- **Alice** changes paragraph 2 to "The sky is blue"
- **Bob** changes paragraph 2 to "The sky is green"

When they reconnect and sync, we have a **conflict**. But how do we know it's a conflict vs. a simple sequential edit?

If Alice edited first, then Bob saw Alice's version and deliberately changed it — that's NOT a conflict (Bob overrode Alice intentionally).

If both edited independently without seeing each other's change — that IS a conflict (neither version is "correct").

**Vector clocks** are the mechanism that lets us distinguish between these two situations. They track "who has seen what" so we can determine if two events are causally related or truly concurrent.

---

## Core Concept Explained Step-by-Step

### The Problem: No Global Clock

```
THE ORDERING PROBLEM:
═══════════════════════════════════════════════════

Node A's clock: 10:00:01.000
Node B's clock: 10:00:01.500  (500ms ahead!)

Event X on Node A at "10:00:01.200"
Event Y on Node B at "10:00:01.300"

Did X happen before Y?
  Using wall clocks: YES (01.200 < 01.300)
  Reality: B's clock is 500ms ahead, so B's "01.300" = real "00.800"
  ACTUAL order: Y happened BEFORE X!

Wall clocks CANNOT be trusted for ordering!
We need LOGICAL clocks.
```

### Step 1: Lamport Timestamps (Simple but Limited)

Invented by Leslie Lamport in 1978. Each event gets a logical counter:

```
LAMPORT TIMESTAMP RULES:
═══════════════════════════════════════════════════

Rule 1: Before each event, increment your counter
Rule 2: When sending a message, include your counter
Rule 3: When receiving, set counter = max(mine, received) + 1

EXAMPLE:
─────────────────────────────────────────────────────────────
Node A (counter=0)         Node B (counter=0)

A does event: counter=1
A sends msg to B ──────────────────▶ B receives
  (carries timestamp 1)               counter = max(0, 1) + 1 = 2

A does event: counter=2              B does event: counter=3

A does event: counter=3              B sends msg to A ─────▶ A receives
                                       (carries timestamp 3)  
                                                            counter = max(3,3)+1 = 4

Timeline:
Node A: [1]───[2]───[3]──────────────────[4]
Node B: ─────────────[2]───[3]──────────────

PROPERTY:
  If event A "happened before" event B → timestamp(A) < timestamp(B)
  
LIMITATION:
  If timestamp(A) < timestamp(B), we CANNOT conclude A happened before B!
  (They might be concurrent events that just got different numbers)
```

### Step 2: Vector Clocks (The Full Solution)

Each node maintains a **vector** (array) with one counter per node in the system:

```
VECTOR CLOCK:
═══════════════════════════════════════════════════

System has 3 nodes: A, B, C
Each node has a vector: [A_counter, B_counter, C_counter]

RULES:
1. Before each local event: increment MY position
2. When sending: include my entire vector
3. When receiving: merge vectors (take max of each position) + increment mine

EXAMPLE:
─────────────────────────────────────────────────────────────

Node A         Node B         Node C
[0,0,0]        [0,0,0]        [0,0,0]

A writes:
[1,0,0]
    │
    │ send to B
    ▼
               [1,1,0]  ← max([0,0,0],[1,0,0])=[1,0,0], then B+1=[1,1,0]
                  │
                  │ send to C
                  ▼
                              [1,1,1]  ← max([0,0,0],[1,1,0])=[1,1,0], then C+1=[1,1,1]

Now A writes again (without seeing B or C's updates):
[2,0,0]

COMPARING:
  Node A: [2,0,0]
  Node C: [1,1,1]
  
  Is [2,0,0] > [1,1,1]? 
    A > C in position A (2>1), but A < C in position B (0<1)
    NEITHER is strictly greater!
    
  Conclusion: These are CONCURRENT events! (Conflict!) 🔥
```

### How to Compare Vector Clocks

```
COMPARISON RULES:
═══════════════════════════════════════════════════

Given vectors V1 and V2:

V1 < V2 (V1 happened before V2):
  Every element in V1 ≤ corresponding element in V2
  AND at least one element is strictly less
  
  Example: [1,2,0] < [1,3,1]  (1≤1, 2≤3, 0≤1) → V1 happened before V2

V1 = V2 (same event):
  All elements are equal
  
  Example: [2,1,3] = [2,1,3]

V1 || V2 (CONCURRENT — conflict!):
  Neither V1 ≤ V2 nor V2 ≤ V1
  (Each is greater in at least one position)
  
  Example: [2,0,0] || [1,1,1]  (2>1 but 0<1) → CONCURRENT!

VISUAL:
─────────────────────────────────────────────────

[1,0,0] ──────▶ [2,0,0] ──────▶ [3,1,0]
    │                                  ↑
    │                                  │
    └──▶ [1,1,0] ──▶ [1,2,0] ─────────┘
              │          
              └──────── This is concurrent with [2,0,0]!
                        [1,2,0] || [2,0,0]
```

---

## How It Works Internally

### Conflict Detection Flow

```
DETECTING AND RESOLVING CONFLICTS:
═══════════════════════════════════════════════════

Client writes "x = hello" to Node A:
  Node A stores: {value: "hello", vector: [1,0,0]}

Client writes "x = world" to Node B (independently!):
  Node B stores: {value: "world", vector: [0,1,0]}

REPLICATION: Node A sends its version to Node B:
  Node B compares:
    Local:    [0,1,0]
    Received: [1,0,0]
    
    Neither is strictly ≥ the other → CONFLICT!
    
RESOLUTION OPTIONS:
─────────────────────────────────────────────────
Option 1: Last-Write-Wins (LWW)
  Compare wall-clock timestamps → one wins, one is lost
  Simple but LOSES DATA!

Option 2: Keep Both (Siblings)
  Store both versions → return to client → client resolves
  "x = {hello OR world} — you decide!"
  
Option 3: Automatic Merge (CRDTs)
  Use data structures that can be merged automatically
  (counters, sets, maps with specific semantics)

Option 4: Application-Specific Logic
  Shopping cart: merge items from both versions
  Wiki: show diff for human resolution
```

### Version Vectors (Practical Vector Clocks)

In practice, systems use **version vectors** — vector clocks indexed by node ID:

```
VERSION VECTOR EXAMPLE (Amazon Dynamo style):
═══════════════════════════════════════════════════

Write 1: Client → Node A
  Key "cart" stored with version: {A:1}
  Value: ["item-1"]

Write 2: Same client → Node A (saw previous)
  Key "cart" stored with version: {A:2}
  Value: ["item-1", "item-2"]

Write 3: Different client → Node B (saw version {A:1} only!)
  Key "cart" stored with version: {A:1, B:1}
  Value: ["item-1", "item-3"]

Now replicate:
  Node A has: {A:2} → ["item-1", "item-2"]
  Node B has: {A:1, B:1} → ["item-1", "item-3"]
  
  Compare: {A:2} vs {A:1, B:1}
  A:2 > A:1 (Node A is ahead in A's counter)
  But B:1 > B:0 (Node B is ahead in B's counter)
  → CONCURRENT! Neither dominates.
  
  Resolution: Merge carts → ["item-1", "item-2", "item-3"]
  New version: {A:2, B:1}
```

### Dotted Version Vectors (Optimization)

```
PROBLEM WITH CLASSIC VERSION VECTORS:
═══════════════════════════════════════════════════

"Sibling explosion" — version vectors grow with conflicts.
If a value has 10 concurrent writes, you store 10 siblings!

DOTTED VERSION VECTORS (used by Riak):
  Track individual "dots" (node:counter pairs) for precise causality.
  
  Instead of: version = {A:5, B:3, C:7}
  Use:        version = {A:5, B:3, C:7}, dot = {B:4}
  
  The "dot" says "THIS specific version was created by B at counter 4"
  This allows more precise conflict detection and fewer false siblings.
```

---

## Code Examples

### Python: Vector Clock Implementation

```python
from typing import Dict, Optional, Tuple, List
from copy import deepcopy
from enum import Enum

class Ordering(Enum):
    BEFORE = "before"        # V1 happened before V2
    AFTER = "after"          # V1 happened after V2
    CONCURRENT = "concurrent" # V1 and V2 are concurrent (conflict!)
    EQUAL = "equal"          # Same version

class VectorClock:
    """A vector clock for tracking causality in distributed systems."""
    
    def __init__(self):
        self.clock: Dict[str, int] = {}
    
    def increment(self, node_id: str):
        """Increment this node's counter (local event)."""
        self.clock[node_id] = self.clock.get(node_id, 0) + 1
    
    def merge(self, other: 'VectorClock'):
        """Merge with another vector clock (receiving a message)."""
        for node, counter in other.clock.items():
            self.clock[node] = max(self.clock.get(node, 0), counter)
    
    def compare(self, other: 'VectorClock') -> Ordering:
        """Compare two vector clocks to determine ordering."""
        all_nodes = set(self.clock.keys()) | set(other.clock.keys())
        
        self_less = False
        other_less = False
        
        for node in all_nodes:
            self_val = self.clock.get(node, 0)
            other_val = other.clock.get(node, 0)
            
            if self_val < other_val:
                self_less = True
            elif self_val > other_val:
                other_less = True
        
        if self_less and not other_less:
            return Ordering.BEFORE  # self happened before other
        elif other_less and not self_less:
            return Ordering.AFTER   # self happened after other
        elif not self_less and not other_less:
            return Ordering.EQUAL
        else:
            return Ordering.CONCURRENT  # conflict!
    
    def copy(self) -> 'VectorClock':
        vc = VectorClock()
        vc.clock = dict(self.clock)
        return vc
    
    def __repr__(self):
        return str(self.clock)


class VersionedValue:
    """A value with its vector clock for conflict detection."""
    
    def __init__(self, value, clock: VectorClock):
        self.value = value
        self.clock = clock
    
    def __repr__(self):
        return f"({self.value}, {self.clock})"


class DynamoStyleStore:
    """Key-value store with vector clocks (like Amazon Dynamo)."""
    
    def __init__(self, node_id: str):
        self.node_id = node_id
        self.data: Dict[str, List[VersionedValue]] = {}  # Key → list of versions
    
    def write(self, key: str, value, context: Optional[VectorClock] = None):
        """
        Write a value with vector clock.
        context = the vector clock from the last READ (to establish causality).
        """
        # Create new clock based on context (or fresh)
        new_clock = context.copy() if context else VectorClock()
        new_clock.increment(self.node_id)
        
        new_version = VersionedValue(value, new_clock)
        
        if key not in self.data:
            self.data[key] = [new_version]
        else:
            # Remove all versions that are dominated by new_clock
            surviving = []
            for existing in self.data[key]:
                ordering = existing.clock.compare(new_clock)
                if ordering == Ordering.CONCURRENT:
                    surviving.append(existing)  # Keep concurrent versions
                elif ordering == Ordering.AFTER:
                    surviving.append(existing)  # Keep newer versions
                # BEFORE or EQUAL → discard (superseded)
            
            surviving.append(new_version)
            self.data[key] = surviving
        
        return new_clock
    
    def read(self, key: str) -> List[VersionedValue]:
        """
        Read returns ALL concurrent versions (siblings).
        Client must resolve conflicts.
        """
        return self.data.get(key, [])


# --- Demonstration ---
print("=== Vector Clock Conflict Detection ===\n")

# Two nodes
store_a = DynamoStyleStore("A")
store_b = DynamoStyleStore("B")

# Client 1 writes to Node A
print("Client 1 writes 'cart = [shoes]' to Node A")
clock_a = store_a.write("cart", ["shoes"])
print(f"  Stored at: {clock_a}\n")

# Simulate: both nodes see this version
store_b.write("cart", ["shoes"], context=None)  # B also has the initial version

# Client 1 writes again to Node A (sees previous version)
print("Client 1 writes 'cart = [shoes, hat]' to Node A (saw previous)")
clock_a2 = store_a.write("cart", ["shoes", "hat"], context=clock_a)
print(f"  Stored at: {clock_a2}")

# Client 2 writes to Node B (only saw FIRST version, not A's update!)
print("Client 2 writes 'cart = [shoes, jacket]' to Node B (saw first version only)")
first_clock = VectorClock()
first_clock.clock = {"A": 1}  # Client 2 only saw version {A:1}
clock_b = store_b.write("cart", ["shoes", "jacket"], context=first_clock)
print(f"  Stored at: {clock_b}")

# Now compare!
print(f"\n--- Conflict Detection ---")
ordering = clock_a2.compare(clock_b)
print(f"Node A version {clock_a2} vs Node B version {clock_b}")
print(f"Relationship: {ordering.value}")

if ordering == Ordering.CONCURRENT:
    print("\n🔥 CONFLICT DETECTED! Both versions must be resolved.")
    print("  Option 1: Merge carts → [shoes, hat, jacket]")
    print("  Option 2: Ask user to choose")
    print("  Option 3: Last-write-wins (loses data)")
```

### Java: Vector Clock with Conflict Resolution

```java
import java.util.*;
import java.util.stream.Collectors;

/**
 * Vector clock implementation for causality tracking.
 * Used in systems like Riak, Voldemort, and Amazon Dynamo.
 */
public class VectorClockSystem {
    
    enum Ordering { BEFORE, AFTER, CONCURRENT, EQUAL }
    
    static class VectorClock {
        private final Map<String, Integer> clock = new HashMap<>();
        
        public void increment(String nodeId) {
            clock.merge(nodeId, 1, Integer::sum);
        }
        
        public void merge(VectorClock other) {
            other.clock.forEach((node, counter) ->
                    clock.merge(node, counter, Math::max));
        }
        
        public Ordering compare(VectorClock other) {
            Set<String> allNodes = new HashSet<>();
            allNodes.addAll(clock.keySet());
            allNodes.addAll(other.clock.keySet());
            
            boolean selfLess = false;
            boolean otherLess = false;
            
            for (String node : allNodes) {
                int selfVal = clock.getOrDefault(node, 0);
                int otherVal = other.clock.getOrDefault(node, 0);
                
                if (selfVal < otherVal) selfLess = true;
                if (selfVal > otherVal) otherLess = true;
            }
            
            if (selfLess && !otherLess) return Ordering.BEFORE;
            if (otherLess && !selfLess) return Ordering.AFTER;
            if (!selfLess && !otherLess) return Ordering.EQUAL;
            return Ordering.CONCURRENT;
        }
        
        public VectorClock copy() {
            VectorClock vc = new VectorClock();
            vc.clock.putAll(this.clock);
            return vc;
        }
        
        @Override
        public String toString() { return clock.toString(); }
    }
    
    record VersionedValue<T>(T value, VectorClock clock) {}
    
    static class ConflictResolver {
        
        /**
         * Resolve concurrent versions of a shopping cart.
         * Strategy: Union merge (combine all items).
         */
        @SuppressWarnings("unchecked")
        public static <T> VersionedValue<List<T>> mergeCarts(
                List<VersionedValue<List<T>>> siblings, String nodeId) {
            
            // Merge all items from all versions (union)
            Set<T> merged = new LinkedHashSet<>();
            VectorClock mergedClock = new VectorClock();
            
            for (VersionedValue<List<T>> sibling : siblings) {
                merged.addAll(sibling.value());
                mergedClock.merge(sibling.clock());
            }
            
            mergedClock.increment(nodeId); // New merged version
            return new VersionedValue<>(new ArrayList<>(merged), mergedClock);
        }
        
        /**
         * Last-write-wins resolution (uses wall clock as tiebreaker).
         * Simple but can lose data!
         */
        public static <T> VersionedValue<T> lastWriteWins(
                List<VersionedValue<T>> siblings) {
            // In real systems, each version has a wall-clock timestamp
            // Here we just pick the last one in the list
            return siblings.get(siblings.size() - 1);
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Vector Clock Demo ===\n");
        
        // Simulate: Two replicas of a shopping cart
        VectorClock clockA = new VectorClock();
        VectorClock clockB = new VectorClock();
        
        // Initial write on Node A
        clockA.increment("A");
        System.out.printf("Write on A: cart=[shoes] clock=%s%n", clockA);
        
        // B sees A's version and adds to it
        clockB.merge(clockA);
        clockB.increment("B");
        System.out.printf("Write on B (saw A): cart=[shoes,hat] clock=%s%n", clockB);
        
        // Meanwhile, A writes again independently (didn't see B's update)
        clockA.increment("A");
        System.out.printf("Write on A (independent): cart=[shoes,jacket] clock=%s%n", clockA);
        
        // Compare versions
        Ordering order = clockA.compare(clockB);
        System.out.printf("%nComparing A=%s vs B=%s%n", clockA, clockB);
        System.out.printf("Result: %s%n", order);
        
        if (order == Ordering.CONCURRENT) {
            System.out.println("\n🔥 CONFLICT! Resolving with union merge...");
            
            List<VersionedValue<List<String>>> siblings = List.of(
                    new VersionedValue<>(List.of("shoes", "jacket"), clockA),
                    new VersionedValue<>(List.of("shoes", "hat"), clockB)
            );
            
            var resolved = ConflictResolver.mergeCarts(siblings, "A");
            System.out.printf("Resolved: %s clock=%s%n", 
                    resolved.value(), resolved.clock());
        }
    }
}
```

---

## Real-World Example

### Amazon Dynamo: Vector Clocks in Production

```
AMAZON DYNAMO (2007 paper):
═══════════════════════════════════════════════════

The shopping cart service uses vector clocks:

Scenario: Customer adds items from phone and laptop simultaneously

Phone (Node A): 
  cart = ["book"]         clock = {A:1}
  cart = ["book", "pen"]  clock = {A:2}

Laptop (Node B, only saw {A:1}):
  cart = ["book", "mug"]  clock = {A:1, B:1}

When syncing:
  {A:2} vs {A:1, B:1} → CONCURRENT!
  
  Dynamo returns BOTH versions to the client app:
  "Here are your cart versions: ['book','pen'] and ['book','mug']"
  
  Application merges: cart = ["book", "pen", "mug"]
  Writes back with merged clock: {A:2, B:1, A:3}  ← resolved!

OPTIMIZATION: Dynamo truncates vector clocks when they get too large
(> 10 entries) using wall-clock timestamps on oldest entries.
This can occasionally create false conflicts but prevents unbounded growth.
```

### Riak: Dotted Version Vectors

```
RIAK's approach:
═══════════════════════════════════════════════════

Riak (now Riak KV) used "dotted version vectors" to reduce false conflicts.

Classic vector clock problem:
  Rapidly updating the same key creates many siblings,
  even when there's no real concurrency (just fast sequential writes).

Dotted version vectors track the SPECIFIC event that created each sibling:
  
  Version 1: value="A", dot=(node1, 5), context={node1:4, node2:3}
  Version 2: value="B", dot=(node2, 4), context={node1:4, node2:3}
  
  The "dot" identifies exactly which write created this version.
  When a new write arrives with context that includes the dot,
  we know it SAW this version and can safely discard it.

Result: Fewer false siblings, more precise conflict detection.
```

### CRDTs: Conflict-Free Replicated Data Types

```
CRDTs — DATA STRUCTURES THAT NEVER CONFLICT:
═══════════════════════════════════════════════════

Instead of detecting and resolving conflicts,
use data structures where ANY merge is valid:

G-Counter (Grow-only counter):
  Each node has its own counter slot.
  Value = sum of all slots.
  Merge = take max of each slot.
  
  Node A: [5, 0, 0]  (Node A incremented 5 times)
  Node B: [0, 3, 0]  (Node B incremented 3 times)
  Merge:  [5, 3, 0]  → Total = 8 ✅ (never conflicts!)

OR-Set (Observed-Remove Set):
  Add/remove operations with unique tags.
  Add always wins over concurrent remove.
  
  Node A: add("item1"), add("item2")
  Node B: remove("item1")  (saw item1 was there)
  
  If concurrent: item1 stays (add wins)
  If B saw A's add first: item1 removed (causal remove)

Used by: Redis CRDTs, Riak, Automerge (collaborative editing)
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using wall clocks for ordering | Clocks drift, NTP adjustments can go backward | Use vector clocks or Lamport timestamps |
| Unbounded vector clock growth | N nodes × M updates = huge vectors | Truncate based on age or cap size (trade-off: occasional false conflicts) |
| Always using Last-Write-Wins | Silently drops data from concurrent writes | Use merge functions or return siblings to clients |
| Ignoring conflict resolution | "Conflicts won't happen in our system" | They WILL at scale. Design your merge strategy upfront. |
| Confusing Lamport timestamps with vector clocks | Lamport can't detect concurrency — only one direction of implication | Use vector clocks when you need to detect concurrent events |
| Not passing context on writes | Write without context = always creates conflict with existing versions | Always read first (get clock), then write with that clock as context |

---

## When to Use / When NOT to Use

### Use Vector Clocks When:
- ✅ AP (Available + Partition-tolerant) systems that allow concurrent writes
- ✅ Multi-master replication where conflicts are possible
- ✅ Collaborative editing systems
- ✅ Shopping carts, document merging, any "merge-able" data

### Use Simpler Approaches When:
- ❌ Single-leader replication (leader determines order — no conflicts)
- ❌ Strong consistency systems (conflicts prevented by locking/consensus)
- ❌ Append-only data (events, logs — no "conflicting updates")
- ❌ Simple counters or sets (CRDTs handle these without vector clocks)

### Use Last-Write-Wins When:
- ✅ Data loss is acceptable (caching, session state)
- ✅ Writes are infrequent and unlikely to be truly concurrent
- ✅ Simplicity is more important than correctness

---

## Key Takeaways

- 🔑 **Wall clocks are unreliable** for ordering events in distributed systems — clock drift makes them untrustworthy.
- 🔑 **Lamport timestamps** give a total order but can't detect concurrency (if L(A) < L(B), A might still be concurrent with B).
- 🔑 **Vector clocks** track full causal history — they can definitively determine if two events are causally ordered or concurrent.
- 🔑 **Concurrent events = conflicts** — the system must resolve them via LWW, merging, sibling storage, or CRDTs.
- 🔑 **Amazon Dynamo** popularized vector clocks for shopping carts — returning siblings to the application for merge decisions.
- 🔑 **CRDTs eliminate conflicts entirely** for specific data structures (counters, sets, registers) by making all merges valid.
- 🔑 **In practice**, most systems use a combination: vector clocks for detection + application-specific merge logic for resolution.

---

## What's Next?

Now that you understand how to detect and resolve conflicts in data, the next chapter covers a fundamental problem: how to distribute data evenly across nodes as the cluster grows and shrinks. **[Consistent Hashing — Distributing Data Evenly](./10-consistent-hashing.md)** explains the elegant algorithm used by virtually every distributed database and cache.
