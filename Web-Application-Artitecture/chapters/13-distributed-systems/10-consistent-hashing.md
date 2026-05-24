# Consistent Hashing — Distributing Data Evenly

> **What you'll learn**: How to distribute data across multiple servers so that adding or removing a server only moves a MINIMAL amount of data — the elegant algorithm behind every major distributed database and cache.

---

## Real-Life Analogy: Assigning Students to Lockers

**Naive approach (modular hashing)**: 100 students, 10 hallways. Student #N goes to hallway N % 10.
- Student 42 → hallway 2, Student 57 → hallway 7, etc.

Now the school adds an 11th hallway. With N % 11:
- Student 42 → hallway 9 (was 2!), Student 57 → hallway 2 (was 7!)
- Almost EVERY student has to move! Chaos on moving day.

**Consistent hashing approach**: Imagine all lockers arranged in a giant circle. Each hallway "owns" a section of the circle. When a new hallway opens, it only takes over the section nearest to it — most students stay where they are. Only ~1/N of students need to move.

---

## Core Concept Explained Step-by-Step

### The Problem with Simple Hashing

```
NAIVE HASH DISTRIBUTION (hash % N):
═══════════════════════════════════════════════════

4 servers: server = hash(key) % 4

  hash("user:1") = 7   → 7 % 4 = 3  → Server 3
  hash("user:2") = 12  → 12 % 4 = 0 → Server 0
  hash("user:3") = 5   → 5 % 4 = 1  → Server 1
  hash("user:4") = 9   → 9 % 4 = 1  → Server 1

Works great! Until...

ADD A SERVER (now 5 servers): server = hash(key) % 5

  hash("user:1") = 7   → 7 % 5 = 2  → Server 2  ← MOVED!
  hash("user:2") = 12  → 12 % 5 = 2 → Server 2  ← MOVED!
  hash("user:3") = 5   → 5 % 5 = 0  → Server 0  ← MOVED!
  hash("user:4") = 9   → 9 % 5 = 4  → Server 4  ← MOVED!

ALL keys moved! With 1 million keys: ~80% remapped.
For a cache: CACHE STAMPEDE! All keys miss simultaneously! 💀
```

### The Consistent Hashing Solution

```
CONSISTENT HASHING:
═══════════════════════════════════════════════════

Imagine a CIRCLE (ring) from 0 to 2^32 (hash space):

         0
         │
    ┌────┴────┐
   ╱           ╲
  │    Hash     │
  │    Ring     │     Servers are placed on the ring
  │  (0 to 2³²)│     using hash(server_name)
   ╲           ╱
    └────┬────┘
         │
       2^32

Step 1: Place SERVERS on the ring
  hash("Server-A") = position 100
  hash("Server-B") = position 400  
  hash("Server-C") = position 700

Step 2: For each KEY, find the NEXT server clockwise
  hash("user:1") = 150 → next server clockwise = Server-B (at 400)
  hash("user:2") = 550 → next server clockwise = Server-C (at 700)
  hash("user:3") = 800 → next server clockwise = Server-A (at 100, wraps!)

            0
            │
      A(100)●
           ╱  ╲
          ╱    ╲  user:1(150) → goes to B
         │      ● B(400)
         │      │
          ╲    ╱  user:2(550) → goes to C
           ╲  ╱
      C(700)●
            │
          user:3(800) → wraps to A
```

### Why It's Better: Adding a Server

```
ADDING SERVER D at position 550:
═══════════════════════════════════════════════════

BEFORE (3 servers):
  A(100), B(400), C(700)
  
  Keys 0-100:   → A
  Keys 101-400: → B
  Keys 401-700: → C
  Keys 701-999: → A (wrap)

AFTER (4 servers, D added at 550):
  A(100), B(400), D(550), C(700)
  
  Keys 0-100:   → A  (unchanged ✅)
  Keys 101-400: → B  (unchanged ✅)
  Keys 401-550: → D  (MOVED from C to D ← only these!)
  Keys 551-700: → C  (unchanged ✅)
  Keys 701-999: → A  (unchanged ✅)

RESULT: Only keys between B(400) and D(550) moved!
        That's roughly 1/N of all keys (1/4 = 25%)
        Compare to naive hash: ~75% would move!

    Before:               After:
    ┌─A─┬─B──┬──C──┐     ┌─A─┬─B──┬─D─┬─C─┐
    0  100  400   700     0  100  400 550 700
                          
    Only this segment moved: ───────►│   │◄──
```

### Virtual Nodes (Vnodes): Solving Uneven Distribution

```
PROBLEM: With few servers, distribution is uneven
═══════════════════════════════════════════════════

3 servers on a ring:
  A at position 100 → owns: 701-100 (400 units)
  B at position 400 → owns: 101-400 (300 units)  
  C at position 700 → owns: 401-700 (300 units)
  
  A has 40% of data, B and C have 30% each. UNBALANCED!

SOLUTION: VIRTUAL NODES (give each server multiple positions)
═══════════════════════════════════════════════════

Each physical server gets ~100-200 virtual positions:

  Server A → hash("A-vnode-0")=50, hash("A-vnode-1")=350, hash("A-vnode-2")=650
  Server B → hash("B-vnode-0")=150, hash("B-vnode-1")=450, hash("B-vnode-2")=850
  Server C → hash("C-vnode-0")=250, hash("C-vnode-1")=550, hash("C-vnode-2")=950

Ring now has 9 points instead of 3:
  50(A) 150(B) 250(C) 350(A) 450(B) 550(C) 650(A) 850(B) 950(C)

Result: Much more even distribution!
  Each server owns ~3 segments scattered around the ring.
  
Benefits of vnodes:
  1. More uniform load distribution
  2. When a server leaves, its load spreads to MANY servers (not just one)
  3. Heterogeneous hardware: powerful server gets more vnodes
```

---

## How It Works Internally

### Hash Ring Data Structure

```
IMPLEMENTATION:
═══════════════════════════════════════════════════

Internal structure: Sorted map of positions → server

  TreeMap<Integer, Server>:
    50   → Server A
    150  → Server B
    250  → Server C
    350  → Server A
    450  → Server B
    550  → Server C
    650  → Server A
    850  → Server B
    950  → Server C

LOOKUP ALGORITHM:
  1. hash = hash_function(key)
  2. Find the first entry in the map where position ≥ hash
  3. If none found (past the last entry), wrap to first entry
  
  Time complexity: O(log N) with balanced tree
  (N = number of virtual nodes)
```

### Replication with Consistent Hashing

```
REPLICATION:
═══════════════════════════════════════════════════

To replicate data on R nodes, walk clockwise from the key's
position and pick the next R DISTINCT physical servers.

Example: Replication factor R = 3

  Key "user:42" hashes to position 300
  
  Ring: ... 250(C) 350(A) 450(B) 550(C) 650(A) ...
  
  Walk clockwise from 300:
    1st: 350 → Server A ✅ (primary)
    2nd: 450 → Server B ✅ (replica 1)
    3rd: 550 → Server C ✅ (replica 2)
  
  Data stored on A, B, and C!
  
  If using vnodes, skip vnodes of same physical server:
    350(A) → 450(B) → 550(C) → three DISTINCT servers ✅
    (Don't count 650(A) — A already has it)

    ●(300) ──▶ A(350) ──▶ B(450) ──▶ C(550)
     key       primary    replica1   replica2
```

### Handling Server Removal

```
SERVER REMOVAL (Server B fails):
═══════════════════════════════════════════════════

Before: A(100) B(400) C(700)

  Keys 101-400 were on Server B.
  
Server B removed:

  Keys 101-400 → now assigned to next clockwise: Server C(700)
  
  Only B's keys are redistributed!
  A's keys stay on A ✅
  C's existing keys stay on C ✅
  C receives B's old keys (about 1/N of total)

With vnodes: B's load distributes to ALL other servers evenly
  (because B had many vnodes scattered around the ring)
  
  B's vnode at 150 → next is C's vnode at 250
  B's vnode at 450 → next is C's vnode at 550
  B's vnode at 850 → next is A's vnode at 950... 
  
  Load spreads evenly! No single server gets overwhelmed.
```

---

## Code Examples

### Python: Consistent Hash Ring

```python
import hashlib
from bisect import bisect_right
from collections import defaultdict
from typing import List, Optional

class ConsistentHashRing:
    """
    Consistent hash ring with virtual nodes.
    Used by systems like Cassandra, DynamoDB, and Memcached.
    """
    
    def __init__(self, num_vnodes: int = 150):
        self.num_vnodes = num_vnodes
        self.ring = {}          # hash_position → physical_node
        self.sorted_keys = []   # Sorted list of positions for binary search
        self.nodes = set()      # Set of physical nodes
    
    def _hash(self, key: str) -> int:
        """Generate a consistent hash value for a key."""
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)
    
    def add_node(self, node: str):
        """Add a physical node with virtual nodes to the ring."""
        self.nodes.add(node)
        
        for i in range(self.num_vnodes):
            vnode_key = f"{node}:vnode:{i}"
            hash_val = self._hash(vnode_key)
            self.ring[hash_val] = node
        
        # Rebuild sorted keys (in production, use a sorted container)
        self.sorted_keys = sorted(self.ring.keys())
        print(f"Added node '{node}' with {self.num_vnodes} vnodes")
    
    def remove_node(self, node: str):
        """Remove a physical node and all its virtual nodes."""
        self.nodes.discard(node)
        
        # Remove all vnodes for this physical node
        keys_to_remove = [k for k, v in self.ring.items() if v == node]
        for key in keys_to_remove:
            del self.ring[key]
        
        self.sorted_keys = sorted(self.ring.keys())
        print(f"Removed node '{node}' ({len(keys_to_remove)} vnodes removed)")
    
    def get_node(self, key: str) -> Optional[str]:
        """Find which node a key belongs to."""
        if not self.ring:
            return None
        
        hash_val = self._hash(key)
        
        # Find first position >= hash_val (clockwise search)
        idx = bisect_right(self.sorted_keys, hash_val)
        
        # Wrap around if past the end
        if idx >= len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def get_nodes_for_key(self, key: str, replicas: int = 3) -> List[str]:
        """Find N distinct physical nodes for replication."""
        if not self.ring:
            return []
        
        hash_val = self._hash(key)
        idx = bisect_right(self.sorted_keys, hash_val)
        
        result = []
        seen_nodes = set()
        
        for i in range(len(self.sorted_keys)):
            position = self.sorted_keys[(idx + i) % len(self.sorted_keys)]
            node = self.ring[position]
            
            if node not in seen_nodes:
                result.append(node)
                seen_nodes.add(node)
                
                if len(result) >= replicas:
                    break
        
        return result
    
    def get_distribution(self, num_keys: int = 10000) -> dict:
        """Check how evenly keys are distributed across nodes."""
        distribution = defaultdict(int)
        
        for i in range(num_keys):
            key = f"key:{i}"
            node = self.get_node(key)
            if node:
                distribution[node] += 1
        
        return dict(distribution)


# --- Demo ---
print("=== Consistent Hash Ring Demo ===\n")

ring = ConsistentHashRing(num_vnodes=150)

# Add initial nodes
for node in ["cache-01", "cache-02", "cache-03"]:
    ring.add_node(node)

# Check distribution
print("\n--- Distribution with 3 nodes (10,000 keys) ---")
dist = ring.get_distribution(10000)
for node, count in sorted(dist.items()):
    print(f"  {node}: {count} keys ({count/100:.1f}%)")

# Look up specific keys
print("\n--- Key lookups ---")
for key in ["user:42", "session:abc", "order:123"]:
    node = ring.get_node(key)
    replicas = ring.get_nodes_for_key(key, replicas=3)
    print(f"  {key} → primary: {node}, replicas: {replicas}")

# Add a 4th node
print("\n--- Adding cache-04 ---")
ring.add_node("cache-04")
dist = ring.get_distribution(10000)
for node, count in sorted(dist.items()):
    print(f"  {node}: {count} keys ({count/100:.1f}%)")

# Check how many keys moved
print("\n--- Movement analysis ---")
moved = 0
for i in range(10000):
    key = f"key:{i}"
    # Compare old vs new assignment (simplified check)
    new_node = ring.get_node(key)
    # In a real system, compare with previous assignment
print("(Approximately 25% of keys move when adding 1 of 4 nodes)")
```

### Java: Consistent Hash Ring

```java
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;

/**
 * Consistent Hash Ring with virtual nodes.
 * Used in production by Cassandra, DynamoDB, and distributed caches.
 */
public class ConsistentHashRing {
    
    private final TreeMap<Long, String> ring = new TreeMap<>();
    private final int numVnodes;
    private final Set<String> physicalNodes = new HashSet<>();
    
    public ConsistentHashRing(int numVnodes) {
        this.numVnodes = numVnodes;
    }
    
    public void addNode(String node) {
        physicalNodes.add(node);
        for (int i = 0; i < numVnodes; i++) {
            long hash = hash(node + ":vnode:" + i);
            ring.put(hash, node);
        }
        System.out.printf("Added '%s' (%d vnodes, ring size: %d)%n", 
                node, numVnodes, ring.size());
    }
    
    public void removeNode(String node) {
        physicalNodes.remove(node);
        ring.entrySet().removeIf(entry -> entry.getValue().equals(node));
        System.out.printf("Removed '%s' (ring size: %d)%n", node, ring.size());
    }
    
    public String getNode(String key) {
        if (ring.isEmpty()) return null;
        
        long hash = hash(key);
        
        // Find first entry >= hash (clockwise)
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
        
        // Wrap around if past the end
        if (entry == null) {
            entry = ring.firstEntry();
        }
        
        return entry.getValue();
    }
    
    public List<String> getReplicaNodes(String key, int replicas) {
        if (ring.isEmpty()) return List.of();
        
        long hash = hash(key);
        List<String> result = new ArrayList<>();
        Set<String> seen = new HashSet<>();
        
        // Walk clockwise from hash position
        NavigableMap<Long, String> tailMap = ring.tailMap(hash, true);
        
        for (String node : tailMap.values()) {
            if (!seen.contains(node)) {
                result.add(node);
                seen.add(node);
                if (result.size() >= replicas) return result;
            }
        }
        
        // Wrap around from the beginning
        for (String node : ring.values()) {
            if (!seen.contains(node)) {
                result.add(node);
                seen.add(node);
                if (result.size() >= replicas) return result;
            }
        }
        
        return result;
    }
    
    public Map<String, Integer> getDistribution(int numKeys) {
        Map<String, Integer> dist = new TreeMap<>();
        for (int i = 0; i < numKeys; i++) {
            String node = getNode("key:" + i);
            dist.merge(node, 1, Integer::sum);
        }
        return dist;
    }
    
    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes());
            // Use first 8 bytes as long
            return ((long)(digest[0] & 0xFF) << 56) |
                   ((long)(digest[1] & 0xFF) << 48) |
                   ((long)(digest[2] & 0xFF) << 40) |
                   ((long)(digest[3] & 0xFF) << 32) |
                   ((long)(digest[4] & 0xFF) << 24) |
                   ((long)(digest[5] & 0xFF) << 16) |
                   ((long)(digest[6] & 0xFF) << 8)  |
                   ((long)(digest[7] & 0xFF));
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
    
    public static void main(String[] args) {
        ConsistentHashRing ring = new ConsistentHashRing(150);
        
        // Add nodes
        ring.addNode("cache-01");
        ring.addNode("cache-02");
        ring.addNode("cache-03");
        
        // Check distribution
        System.out.println("\n--- Distribution (10,000 keys) ---");
        ring.getDistribution(10000).forEach((node, count) ->
                System.out.printf("  %s: %d keys (%.1f%%)%n", node, count, count / 100.0));
        
        // Lookup with replicas
        System.out.println("\n--- Replica lookup ---");
        for (String key : List.of("user:42", "session:abc", "order:123")) {
            List<String> replicas = ring.getReplicaNodes(key, 3);
            System.out.printf("  %s → %s%n", key, replicas);
        }
        
        // Add a 4th node
        System.out.println();
        ring.addNode("cache-04");
        System.out.println("\n--- Distribution after adding cache-04 ---");
        ring.getDistribution(10000).forEach((node, count) ->
                System.out.printf("  %s: %d keys (%.1f%%)%n", node, count, count / 100.0));
    }
}
```

---

## Infrastructure Examples

### Cassandra: Token Ring

```
CASSANDRA'S CONSISTENT HASH RING:
═══════════════════════════════════════════════════

Cassandra uses a ring with token ranges (Murmur3 hash):
Hash range: -2^63 to +2^63

  Node 1: token = -4611686018427387904
  Node 2: token = 0
  Node 3: token = 4611686018427387904
  
  ┌─────────────────────────────────────────────┐
  │     Cassandra Ring (3 nodes, RF=3)          │
  │                                             │
  │         Node 1 (-4.6×10^18)                │
  │            ╱      ╲                         │
  │           ╱        ╲                        │
  │     Node 3          Node 2                  │
  │   (4.6×10^18)        (0)                   │
  │                                             │
  │  Each node owns 1/3 of the token range      │
  │  With RF=3: each node stores ALL data       │
  │  (every key replicated to 3 nodes)          │
  └─────────────────────────────────────────────┘

Adding a node: Cassandra streams data from neighbors automatically.
  New node announces its token → takes ownership of a range →
  receives data from the previous owner of that range.
```

### Memcached: Client-Side Consistent Hashing

```
MEMCACHED CLIENT LIBRARY (e.g., libmemcached):
═══════════════════════════════════════════════════

Memcached servers don't communicate with each other!
The CLIENT implements consistent hashing:

  Client has list of servers: [mc1, mc2, mc3, mc4]
  Client builds hash ring locally
  
  cache.set("user:42", data)
    → Client hashes "user:42"
    → Finds mc2 on the ring
    → Sends SET to mc2 directly
  
  cache.get("user:42")
    → Client hashes "user:42" (same hash = same server)
    → Sends GET to mc2 directly

If mc2 dies:
  Client removes mc2 from local ring
  Only mc2's keys are redistributed
  Other ~75% of keys still hit correct server!
  
This is why consistent hashing was invented! (1997, Karger et al.)
Original motivation: web caches.
```

---

## Real-World Example

### Amazon DynamoDB: Partition Key Hashing

```
DYNAMODB PARTITION ASSIGNMENT:
═══════════════════════════════════════════════════

DynamoDB uses consistent hashing internally:

  Table: "Users"
  Partition Key: "userId"
  
  hash("user-001") → Partition A (stored on storage node group 1)
  hash("user-002") → Partition B (stored on storage node group 2)
  hash("user-003") → Partition A (same partition as user-001)
  
When table grows:
  DynamoDB SPLITS partitions automatically
  Uses consistent hashing to minimize data movement
  
  Before: Partition A handles hash range [0, 1000]
  After:  Partition A handles [0, 500]
          Partition A' handles [501, 1000]  ← new partition!
  
  Only ~half of Partition A's data moves to A'.
  All other partitions unaffected.

Hot partition detection:
  If hash("celebrity-user") gets too much traffic,
  DynamoDB can split THAT partition further (adaptive capacity).
```

### Discord: Consistent Hashing for Guild Routing

```
DISCORD'S MESSAGE ROUTING:
═══════════════════════════════════════════════════

Discord routes messages to the correct "guild process" 
using consistent hashing on guild_id:

  hash(guild_id) → determines which server handles that guild
  
  Millions of guilds across thousands of servers
  When a server is added/removed:
    Only guilds near it on the ring are moved
    Other guilds continue uninterrupted
  
  Result: Server changes affect < 5% of active guilds
          (vs. 80%+ with naive modulo hashing)
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Too few virtual nodes | Uneven distribution (some servers get 2x the load) | Use 100-200 vnodes per physical server |
| Not accounting for server capacity | All servers get equal vnodes but have different CPU/RAM | Assign more vnodes to more powerful servers |
| Hash function with poor distribution | Clustering of keys on few servers | Use MD5, SHA-1, or Murmur3 for uniform distribution |
| Ignoring replication in ring walk | Picking adjacent vnodes that map to same physical server | Skip vnodes of the same physical server when selecting replicas |
| Not handling "hotspots" | One key gets extreme traffic regardless of distribution | Add application-level sharding or caching for hot keys |
| Rebuilding ring on every change | Expensive for large rings | Incremental add/remove operations on sorted structure |

---

## When to Use / When NOT to Use

### Use Consistent Hashing When:
- ✅ Distributed caches (Memcached, Redis Cluster)
- ✅ Distributed databases (Cassandra, DynamoDB, Riak)
- ✅ Load balancing with sticky sessions
- ✅ Content delivery networks (map content to edge servers)
- ✅ Any system where nodes join/leave frequently

### Do NOT Use When:
- ❌ Data fits on a single machine (no distribution needed)
- ❌ You need range queries (consistent hashing scatters sequential keys)
- ❌ Server set NEVER changes (simple modulo hash works fine)
- ❌ You need ordered data distribution (use range partitioning instead)

---

## Key Takeaways

- 🔑 **Consistent hashing** maps both keys and servers to a ring, minimizing data movement when servers are added/removed.
- 🔑 **With N servers**, adding one only moves ~1/N of keys (vs. ~(N-1)/N with naive modulo).
- 🔑 **Virtual nodes** (vnodes) solve the uneven distribution problem by giving each server many positions on the ring.
- 🔑 **Replication** works by walking clockwise on the ring and picking the next R distinct physical servers.
- 🔑 **Used everywhere**: Cassandra, DynamoDB, Memcached, Redis Cluster, Chord DHT, Akamai CDN.
- 🔑 **O(log N) lookup time** using a sorted tree structure (TreeMap/SortedList of ring positions).
- 🔑 **Originally invented for web caching** (1997) — now fundamental to virtually all distributed systems.

---

## What's Next?

Consistent hashing helps distribute data, but how do all nodes in a cluster learn about each other and share state? The next chapter covers **[Gossip Protocol — How Nodes Talk to Each Other](./11-gossip-protocol.md)** — the epidemic-style communication that keeps distributed clusters in sync.
