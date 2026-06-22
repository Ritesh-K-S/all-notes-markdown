# 🔪 Chapter 7.1 — Database Sharding — Horizontal Scaling

> **Level:** 🔴 Advanced | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 1.6 (CAP Theorem), Chapter 1.7 (Indexing), Chapter 2.2 (Replication)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why** sharding exists and when you truly need it
- Master **3 sharding strategies** (Hash, Range, Directory) with trade-offs
- Design a **shard key** that won't destroy your application at scale
- Handle the brutal realities: **resharding**, **cross-shard queries**, **hotspots**
- See how **real companies** (Instagram, Discord, Notion, Uber) shard their databases
- Know the **difference between sharding in SQL vs NoSQL** databases
- Avoid the **Top 10 sharding mistakes** that bring systems down at 3 AM

---

## 🧠 The Problem — Why Sharding Exists

### The Story of GrowthDB Corp.

```
Year 1: "Our PostgreSQL handles everything!"
         → 50 GB of data, 1,000 queries/sec
         → Single server, life is beautiful ☀️

Year 2: "Let's add read replicas!"
         → 500 GB, 10,000 queries/sec
         → 1 primary + 3 replicas = reads are fast

Year 3: "Writes are killing us..."
         → 5 TB, 50,000 queries/sec
         → All writes go to ONE server
         → Vertical scaling? Max CPU = 128 cores, Max RAM = 2 TB
         → 💀 We've hit the ceiling.

Year 4: "We need to SPLIT the data across multiple servers"
         → Welcome to SHARDING.
```

> **Sharding = Splitting one large database into multiple smaller databases (shards), each holding a SUBSET of the data, running on SEPARATE servers.**

---

## 📐 Vertical Scaling vs Horizontal Scaling

```
                    VERTICAL SCALING (Scale UP)
                    ═══════════════════════════
                    
                    ┌────────────────────┐
                    │                    │
                    │   BIGGER SERVER    │
                    │                    │
                    │  128 cores CPU     │
                    │  2 TB RAM          │
                    │  100 TB NVMe SSD   │
                    │                    │
                    │  Cost: $50K/month  │
                    │                    │
                    │  ⚠️ STILL ONE      │
                    │    SINGLE POINT    │
                    │    OF FAILURE      │
                    └────────────────────┘

                    HORIZONTAL SCALING (Scale OUT)  ← SHARDING
                    ═══════════════════════════════
                    
        ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │ Shard 4  │
        │          │  │          │  │          │  │          │
        │ Users    │  │ Users    │  │ Users    │  │ Users    │
        │ A - F    │  │ G - L    │  │ M - R    │  │ S - Z    │
        │          │  │          │  │          │  │          │
        │ 16 cores │  │ 16 cores │  │ 16 cores │  │ 16 cores │
        │ 64GB RAM │  │ 64GB RAM │  │ 64GB RAM │  │ 64GB RAM │
        └──────────┘  └──────────┘  └──────────┘  └──────────┘
        
        Cost: $2K each = $8K/month total
        ✅ No single point of failure
        ✅ Add more shards as needed
        ✅ Each shard is independent
```

### When Do You Actually Need Sharding?

```
┌────────────────────────────────────────────────────────────────────┐
│  🚨 SHARDING IS A LAST RESORT — NOT A FIRST CHOICE               │
│                                                                    │
│  TRY THESE FIRST (in order):                                      │
│                                                                    │
│  1. ✅ Optimize queries & indexes                                  │
│  2. ✅ Add connection pooling (PgBouncer, ProxySQL)                │
│  3. ✅ Add read replicas for read-heavy workloads                  │
│  4. ✅ Vertical scaling (bigger machine)                           │
│  5. ✅ Caching layer (Redis, Memcached)                            │
│  6. ✅ Archive old data / Partitioning                             │
│  7. ✅ Denormalize hot paths                                       │
│  8. 🔪 Sharding (only when all above fail)                         │
│                                                                    │
│  "Premature sharding is the root of all database evil."            │
└────────────────────────────────────────────────────────────────────┘
```

💡 **Real Talk:** Instagram ran on a single PostgreSQL server for their first **25 million users**. They sharded only when they had to. Don't shard on day one.

---

## 🗂️ Sharding Architectures

### The Big Picture

```
                        Application Layer
                              │
                              ▼
                    ┌────────────────────┐
                    │   Shard Router /   │
                    │   Proxy / SDK      │
                    │                    │
                    │  "Which shard has  │
                    │   this user's      │
                    │   data?"           │
                    └────────┬───────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Shard 1  │  │ Shard 2  │  │ Shard 3  │
        │          │  │          │  │          │
        │  Users   │  │  Users   │  │  Users   │
        │  1-1M    │  │  1M-2M   │  │  2M-3M   │
        │          │  │          │  │          │
        │ + Replica│  │ + Replica│  │ + Replica│
        └──────────┘  └──────────┘  └──────────┘
```

---

## 🎲 Strategy 1: Hash-Based Sharding

> **The most common approach — used by MongoDB, Cassandra, DynamoDB, CockroachDB**

### How It Works

```
Step 1: Pick a shard key (e.g., user_id)
Step 2: Hash it → hash(user_id) = some number
Step 3: Modulo by number of shards → shard = hash(user_id) % N

                    hash("user_42") = 8274619274
                    8274619274 % 4 = 2
                    → Goes to Shard 2!

  ┌──────────────────────────────────────────────────────────────┐
  │                    Hash Function                              │
  │                                                               │
  │  user_id = 42  ──► hash(42) = 8274619274 ──► 8274619274 % 4  │
  │                                                    = 2        │
  │  user_id = 99  ──► hash(99) = 3847261589 ──► 3847261589 % 4  │
  │                                                    = 1        │
  │  user_id = 7   ──► hash(7)  = 6182937461 ──► 6182937461 % 4  │
  │                                                    = 1        │
  │  user_id = 200 ──► hash(200)= 1029384756 ──► 1029384756 % 4  │
  │                                                    = 0        │
  └──────────────────────────────────────────────────────────────┘

        Shard 0          Shard 1          Shard 2         Shard 3
      ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
      │ user_200 │    │ user_99  │    │ user_42  │    │          │
      │          │    │ user_7   │    │          │    │          │
      └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

### Pros & Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Even data distribution | Range queries are IMPOSSIBLE efficiently |
| Simple to implement | Adding/removing shards = rehash EVERYTHING |
| No hotspots (with good hash) | Can't query "all users named A-F" |
| Predictable routing | Need consistent hashing to mitigate resharding |

### The Resharding Nightmare

```
PROBLEM: You have 4 shards and need to add a 5th

Before (4 shards):  hash(key) % 4 = shard_number
After  (5 shards):  hash(key) % 5 = DIFFERENT shard_number!

  Key    hash(key)   % 4    % 5
  ────   ─────────   ───    ───
  42     8274619274   2      4    ← MOVED! 😱
  99     3847261589   1      4    ← MOVED! 😱
  7      6182937461   1      1    ← Same (lucky)
  200    1029384756   0      1    ← MOVED! 😱

  ~75% of ALL data needs to be relocated! 💀
```

### Solution: Consistent Hashing

```
         Consistent Hashing Ring
         ═══════════════════════

                    0°
                    │
           ┌───────┼───────┐
          /        │        \
        /          │          \
      /     S1●    │            \
    /    (0-90°)   │             \
   │               │              │
  270°─────────────┼──────────────90°
   │               │              │
    \              │     ●S2    /
      \            │  (90-180°)/
        \          │        /
          \        │      /
           └───────┼───┘
                   │
                  180°
                   ●S3
              (180-270°)

  Adding Shard S4 at 135°:
  → Only keys between 90° and 135° move to S4
  → That's ~12.5% of data, NOT 75%! ✅

  Virtual Nodes: Each shard gets multiple positions
  on the ring for better distribution:

  S1: {15°, 95°, 200°, 310°}    ← 4 virtual nodes
  S2: {45°, 130°, 230°, 340°}
  S3: {75°, 170°, 270°, 350°}
```

### Implementation — Consistent Hashing in Code

```python
import hashlib
from bisect import bisect_right

class ConsistentHashRing:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.ring = {}
        self.sorted_keys = []
        self.virtual_nodes = virtual_nodes
        
        if nodes:
            for node in nodes:
                self.add_node(node)
    
    def _hash(self, key):
        """Generate hash for a key"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def add_node(self, node):
        """Add a node with virtual nodes for better distribution"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:vn{i}"
            hash_val = self._hash(virtual_key)
            self.ring[hash_val] = node
            self.sorted_keys.append(hash_val)
        self.sorted_keys.sort()
    
    def remove_node(self, node):
        """Remove a node — only its keys get redistributed"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:vn{i}"
            hash_val = self._hash(virtual_key)
            del self.ring[hash_val]
            self.sorted_keys.remove(hash_val)
    
    def get_node(self, key):
        """Find which node a key belongs to"""
        if not self.ring:
            return None
        hash_val = self._hash(key)
        # Find the first node clockwise on the ring
        idx = bisect_right(self.sorted_keys, hash_val)
        if idx == len(self.sorted_keys):
            idx = 0  # Wrap around
        return self.ring[self.sorted_keys[idx]]

# Usage
ring = ConsistentHashRing(["shard-1", "shard-2", "shard-3"])

print(ring.get_node("user:42"))    # → "shard-2"
print(ring.get_node("user:99"))    # → "shard-1"
print(ring.get_node("order:500"))  # → "shard-3"

# Add a new shard — only ~1/N keys relocate!
ring.add_node("shard-4")
print(ring.get_node("user:42"))    # Might still be "shard-2" ✅
```

---

## 📊 Strategy 2: Range-Based Sharding

> **Split data by value ranges — great for time-series and geographic data**

### How It Works

```
  ┌─────────────────────────────────────────────────────┐
  │              Range-Based Sharding                    │
  │                                                      │
  │  Shard Key: created_date                             │
  │                                                      │
  │  Shard 1: Jan 2024 – Jun 2024                       │
  │  Shard 2: Jul 2024 – Dec 2024                       │
  │  Shard 3: Jan 2025 – Jun 2025                       │
  │  Shard 4: Jul 2025 – Dec 2025                       │
  │                                                      │
  │  ─────── OR ───────                                  │
  │                                                      │
  │  Shard Key: region                                   │
  │                                                      │
  │  Shard US:     All North American users              │
  │  Shard EU:     All European users                    │
  │  Shard APAC:   All Asia-Pacific users                │
  └─────────────────────────────────────────────────────┘

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Shard 1  │    │ Shard 2  │    │ Shard 3  │    │ Shard 4  │
  │          │    │          │    │          │    │          │
  │ IDs 1-   │    │ IDs      │    │ IDs      │    │ IDs      │
  │ 1,000,000│    │ 1M - 2M  │    │ 2M - 3M  │    │ 3M - 4M  │
  │          │    │          │    │          │    │          │
  │ 🥶 Cold  │    │ 🥶 Cold  │    │ 🔥 Warm  │    │ 🔥 HOT   │
  │ (old     │    │          │    │          │    │ (newest  │
  │  data)   │    │          │    │          │    │  data)   │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
       ↑                                              ↑
   Almost no              All new writes land here =
   traffic                       HOTSPOT! 🔥🔥🔥
```

### Pros & Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Range queries are FAST | **Hotspots** — new data hits one shard |
| Easy to understand | Uneven data distribution |
| Natural for time-series data | The "latest" shard is always overloaded |
| Efficient for geographic partitioning | Need manual rebalancing |

### When Range Sharding Works Perfectly

```
✅ GREAT FOR:
   • Logging systems (shard by month/year)
   • Multi-tenant SaaS (shard by tenant_id ranges)
   • Geographic data (shard by region)
   • Historical archives (shard by date)

❌ TERRIBLE FOR:
   • Social media feeds (latest shard = all the traffic)
   • User-facing apps with monotonic IDs
   • Any workload where "newest data" gets most traffic
```

---

## 📖 Strategy 3: Directory-Based Sharding (Lookup Table)

> **A lookup service tells you exactly where each piece of data lives**

### How It Works

```
                    ┌─────────────────────────┐
                    │     Lookup Service       │
                    │     (Directory DB)        │
                    │                          │
                    │  tenant_id → shard       │
                    │  ─────────────────       │
                    │  acme_corp  → shard_1    │
                    │  globex     → shard_2    │
                    │  initech   → shard_1    │
                    │  umbrella   → shard_3    │
                    │  stark_ind  → shard_2    │
                    │  wayne_ent  → shard_3    │
                    └───────────┬──────────────┘
                                │
                    ┌───────────┼───────────┐
                    │           │           │
                    ▼           ▼           ▼
              ┌──────────┐┌──────────┐┌──────────┐
              │ Shard 1  ││ Shard 2  ││ Shard 3  │
              │          ││          ││          │
              │ acme_corp││ globex   ││ umbrella │
              │ initech  ││ stark_ind││ wayne_ent│
              └──────────┘└──────────┘└──────────┘
```

### Pros & Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Maximum flexibility | Lookup service = **single point of failure** |
| Can move data between shards easily | Extra network hop for EVERY query |
| Handles uneven tenants well | Lookup table itself can become bottleneck |
| No reshashing needed | More complex infrastructure |
| Can co-locate related data | Must cache lookups aggressively |

---

## ⚡ Sharding Comparison — All 3 Strategies

```
┌───────────────┬──────────────────┬──────────────────┬──────────────────┐
│   Criteria    │   Hash-Based     │   Range-Based    │   Directory      │
├───────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Distribution  │ ✅ Even          │ ⚠️ Can be uneven │ ✅ Configurable  │
│ Range Queries │ ❌ Scatter-gather│ ✅ Efficient     │ ⚠️ Depends       │
│ Add Shards    │ ⚠️ Rehash/       │ ✅ Easy split    │ ✅ Just update   │
│               │   Consistent hash│                  │   lookup table   │
│ Hotspots      │ ✅ Rare          │ ❌ Common        │ ✅ Manageable    │
│ Complexity    │ 🟡 Medium        │ 🟢 Low           │ 🔴 High          │
│ Flexibility   │ 🟡 Medium        │ 🟡 Medium        │ 🟢 Maximum       │
│ Used By       │ MongoDB, Cassandra│ HBase, Bigtable │ Multi-tenant SaaS│
│               │ DynamoDB, CockroachDB│ Time-series DBs│ Custom solutions │
└───────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## 🔑 The Art of Choosing a Shard Key

> **The shard key is the MOST important decision in sharding. Get it wrong, and you'll suffer for years.**

### The Perfect Shard Key

```
┌────────────────────────────────────────────────────────────────────┐
│                 THE PERFECT SHARD KEY HAS:                         │
│                                                                    │
│  1. HIGH CARDINALITY                                               │
│     → Many unique values (user_id ✅, country ❌ too few values)  │
│                                                                    │
│  2. EVEN DISTRIBUTION                                              │
│     → Data spreads evenly (user_id ✅, created_date ❌ hotspots)  │
│                                                                    │
│  3. QUERY ISOLATION                                                │
│     → Most queries can target ONE shard, not all                   │
│     → user_id is great if most queries are "get user X's data"     │
│                                                                    │
│  4. NO MONOTONIC INCREASE                                          │
│     → Auto-increment IDs ❌ (all new data → same shard)           │
│     → UUIDs ✅ (random distribution)                              │
│     → Timestamps ❌ (unless combined with other fields)           │
│                                                                    │
│  5. IMMUTABLE                                                      │
│     → The value should NEVER change (email ❌, user_id ✅)        │
│     → Changing shard key = move data between shards = nightmare    │
└────────────────────────────────────────────────────────────────────┘
```

### Shard Key Examples — Good vs Bad

```
┌──────────────────────────────────────────────────────────────────┐
│  APPLICATION: E-Commerce Platform                                 │
│                                                                   │
│  ❌ BAD Shard Keys:                                               │
│                                                                   │
│  • country        → Only ~200 values, USA gets 60% of traffic    │
│  • created_date   → Newest shard = hotspot                       │
│  • auto_inc_id    → Monotonic, latest shard overloaded           │
│  • status         → Only 5 values (pending, shipped, etc.)       │
│                                                                   │
│  ✅ GOOD Shard Keys:                                              │
│                                                                   │
│  • user_id        → High cardinality, queries are per-user       │
│  • order_id (UUID)→ Random distribution, unique                  │
│  • {user_id, order_id} → Compound key, co-locates user's orders │
│                                                                   │
│  🏆 BEST for this case:                                           │
│  • user_id → Because 90% of queries are:                         │
│    "Show me MY orders", "Update MY cart", "MY payment history"    │
│    All go to ONE shard! ✅                                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 💣 The Hardest Problems in Sharding

### Problem 1: Cross-Shard Queries (Scatter-Gather)

```
  Query: "Show the top 10 best-selling products across ALL users"

  This query doesn't have a shard key! It needs data from ALL shards.

  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Shard 1  │    │ Shard 2  │    │ Shard 3  │
  │          │    │          │    │          │
  │ "My top  │    │ "My top  │    │ "My top  │
  │  10..."  │    │  10..."  │    │  10..."  │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
       └───────────────┼───────────────┘
                       │
                       ▼
              ┌────────────────┐
              │   Aggregator   │
              │                │
              │ Merge 30 rows  │
              │ Re-sort        │
              │ Return top 10  │
              └────────────────┘

  Problems:
  • N shards = N queries (slow!)
  • Aggregation in application layer
  • Sorting across shards is complex
  • Pagination becomes a nightmare (OFFSET across shards?)
```

**Solutions:**

```
1. DENORMALIZE: Store a "global top products" table separately
   → Updated asynchronously via events

2. CQRS: Separate read model that aggregates across shards
   → Read from a denormalized view, write to shards

3. SECONDARY INDEX SERVICE: 
   → Elasticsearch indexes data from all shards
   → Query ES for search, fetch from shards for detail

4. CAREFUL SCHEMA DESIGN:
   → Co-locate related data on the same shard
   → {user_id, order_id} on same shard = JOINs work!
```

### Problem 2: Cross-Shard Joins

```sql
-- This simple query becomes a NIGHTMARE with sharding:

SELECT u.name, o.total, p.product_name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN products p ON o.product_id = p.id
WHERE u.id = 42;

-- If sharded by user_id:
-- users(42)  → Shard 2 ✅
-- orders(user_id=42) → Shard 2 ✅ (co-located!)
-- products(id=...) → Could be on ANY shard ❌

-- Solutions:
-- 1. Replicate the 'products' table to ALL shards (reference/lookup table)
-- 2. Denormalize: store product_name directly in orders table
-- 3. Fetch from products shard separately (application-level join)
```

### Problem 3: Cross-Shard Transactions

```
  Transfer $500 from User A (Shard 1) to User B (Shard 3)

  ┌──────────┐                        ┌──────────┐
  │ Shard 1  │                        │ Shard 3  │
  │          │                        │          │
  │ User A   │   ❓ How to make      │ User B   │
  │ -$500    │   this ATOMIC?         │ +$500    │
  └──────────┘                        └──────────┘

  Options:
  ┌──────────────────────────────────────────────────────┐
  │ 1. Two-Phase Commit (2PC)                            │
  │    → Coordinator asks both shards to PREPARE         │
  │    → If both say YES → COMMIT on both                │
  │    → If any says NO → ROLLBACK on both               │
  │    → Problem: SLOW, coordinator = SPOF               │
  │                                                      │
  │ 2. Saga Pattern (Compensating Transactions)          │
  │    → Debit Shard 1 → Credit Shard 3                  │
  │    → If Credit fails → Run "Refund" on Shard 1       │
  │    → Eventually consistent, no distributed lock      │
  │                                                      │
  │ 3. Co-locate the data!                               │
  │    → Put both users on the same shard                 │
  │    → Not always possible, but ideal                   │
  └──────────────────────────────────────────────────────┘
```

### Problem 4: Auto-Increment IDs Across Shards

```
  Problem: Two shards both generate id = 1, id = 2, id = 3...
  
  ┌──────────────────────────────────────────────────────────────┐
  │  SOLUTIONS:                                                   │
  │                                                               │
  │  1. UUID/ULID                                                 │
  │     → Globally unique, no coordination                        │
  │     → "550e8400-e29b-41d4-a716-446655440000"                 │
  │     → ⚠️ 128-bit, bad for B+Tree clustering                 │
  │                                                               │
  │  2. Snowflake IDs (Twitter's approach)                       │
  │     → 64-bit integer: timestamp + machine_id + sequence       │
  │     → Sortable, unique, compact                               │
  │     │ 41 bits: timestamp │ 10 bits: machine │ 12 bits: seq │ │
  │     → Example: 1541815603606036480                            │
  │                                                               │
  │  3. Ticket Servers (Flickr's approach)                       │
  │     → Centralized ID generator                                │
  │     → Shard 1: odd IDs (1, 3, 5, 7...)                      │
  │     → Shard 2: even IDs (2, 4, 6, 8...)                     │
  │     → Simple but central service = bottleneck                 │
  │                                                               │
  │  4. ULID (Universally Unique Lexicographically Sortable ID) │
  │     → Like UUID but sortable by time                          │
  │     → "01ARZ3NDEKTSV4RRFFQ69G5FAV"                          │
  │     → Best of both worlds ✅                                  │
  └──────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Sharding in Real Databases

### MongoDB Sharding

```
  MongoDB's built-in sharding architecture:

  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │  mongos    │  │  mongos    │  │  mongos    │
  │  (Router)  │  │  (Router)  │  │  (Router)  │
  └─────┬──────┘  └─────┬──────┘  └──────┬─────┘
        │               │               │
        └───────────────┼───────────────┘
                        │
            ┌───────────┼───────────┐
            │           │           │
            ▼           ▼           ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │  Config    │ │  Config    │ │  Config    │
     │  Server    │ │  Server    │ │  Server    │
     │ (metadata) │ │ (replica)  │ │ (replica)  │
     └────────────┘ └────────────┘ └────────────┘
            │           │           │
            ▼           ▼           ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │  Shard 1   │ │  Shard 2   │ │  Shard 3   │
     │ (Replica   │ │ (Replica   │ │ (Replica   │
     │  Set)      │ │  Set)      │ │  Set)      │
     └────────────┘ └────────────┘ └────────────┘
```

```javascript
// MongoDB: Enable sharding on a database
sh.enableSharding("ecommerce")

// Shard a collection with HASHED shard key
sh.shardCollection("ecommerce.orders", { user_id: "hashed" })

// Shard with RANGE-based key
sh.shardCollection("ecommerce.logs", { created_at: 1 })

// Compound shard key (best for query isolation)
sh.shardCollection("ecommerce.orders", { user_id: 1, order_date: 1 })

// Check shard distribution
db.orders.getShardDistribution()
// Output:
// Shard shard-1: 33.2% data, 12,847 docs
// Shard shard-2: 33.5% data, 12,991 docs
// Shard shard-3: 33.3% data, 12,862 docs  ← Balanced! ✅
```

### PostgreSQL — Application-Level Sharding with Citus

```sql
-- Citus extension: Distributed PostgreSQL

-- Install Citus extension
CREATE EXTENSION citus;

-- Add worker nodes
SELECT citus_add_node('shard1.example.com', 5432);
SELECT citus_add_node('shard2.example.com', 5432);
SELECT citus_add_node('shard3.example.com', 5432);

-- Create a distributed table (sharded by user_id)
CREATE TABLE orders (
    order_id    BIGINT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    product_id  INT,
    amount      DECIMAL(10,2),
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Distribute the table across shards
SELECT create_distributed_table('orders', 'user_id');

-- Create a REFERENCE table (replicated to all shards for JOINs)
CREATE TABLE products (
    product_id  INT PRIMARY KEY,
    name        VARCHAR(255),
    price       DECIMAL(10,2)
);
SELECT create_reference_table('products');

-- Now this JOIN works efficiently!
-- Orders are on the shard, products are replicated everywhere
SELECT o.order_id, p.name, o.amount
FROM orders o
JOIN products p ON o.product_id = p.product_id
WHERE o.user_id = 42;  -- Routes to ONE shard ✅
```

### MySQL — Vitess (YouTube's Sharding Solution)

```yaml
# Vitess VSchema — defines sharding rules
{
  "sharded": true,
  "vindexes": {
    "hash": {
      "type": "hash"
    }
  },
  "tables": {
    "users": {
      "column_vindexes": [
        {
          "column": "user_id",
          "name": "hash"
        }
      ]
    },
    "orders": {
      "column_vindexes": [
        {
          "column": "user_id",
          "name": "hash"
        }
      ]
    }
  }
}
```

```sql
-- Vitess: Queries look like normal MySQL!
-- Vitess proxy (vtgate) handles routing transparently

SELECT * FROM users WHERE user_id = 42;
-- → vtgate routes to correct shard automatically

-- Cross-shard query (scatter-gather):
SELECT COUNT(*) FROM users;
-- → vtgate queries ALL shards, aggregates results
```

### Cassandra — Built-In Sharding (Partitioning)

```cql
-- Cassandra uses partition keys for automatic sharding

CREATE TABLE orders (
    user_id     UUID,
    order_date  TIMESTAMP,
    order_id    UUID,
    amount      DECIMAL,
    PRIMARY KEY ((user_id), order_date, order_id)
);
-- (user_id) = partition key → determines which node stores the data
-- order_date, order_id = clustering keys → sort within partition

-- This query hits ONE node (partition-aware) ✅
SELECT * FROM orders WHERE user_id = 'abc-123';

-- This query hits ONE node, range within partition ✅
SELECT * FROM orders 
WHERE user_id = 'abc-123' 
AND order_date > '2025-01-01';

-- This query hits ALL nodes (full cluster scan) ❌ AVOID!
SELECT * FROM orders WHERE amount > 100;
-- Solution: Add "ALLOW FILTERING" (don't!) or create a materialized view
```

---

## 🏢 Real-World Sharding Case Studies

### Instagram (PostgreSQL Sharding)

```
┌─────────────────────────────────────────────────────────────┐
│  INSTAGRAM'S APPROACH:                                       │
│                                                              │
│  Scale: Billions of photos, 2+ billion users                │
│  Database: PostgreSQL (sharded)                              │
│                                                              │
│  Shard Key: user_id                                         │
│  Strategy: Logical sharding → thousands of logical shards   │
│            mapped to fewer physical servers                   │
│                                                              │
│  ID Generation: Modified Snowflake                           │
│  │ 41 bits: timestamp │ 13 bits: logical shard │ 10 bits: seq│
│                                                              │
│  Why user_id?                                                │
│  → "Show me MY feed" = one shard                            │
│  → "Show me MY photos" = one shard                          │
│  → "Show me MY likes" = one shard                           │
│  → 95% of queries are user-scoped!                          │
│                                                              │
│  Reference tables replicated everywhere:                     │
│  → hashtags, locations, filters                              │
└─────────────────────────────────────────────────────────────┘
```

### Discord (Cassandra → ScyllaDB)

```
┌─────────────────────────────────────────────────────────────┐
│  DISCORD'S MESSAGE STORAGE:                                  │
│                                                              │
│  Scale: Trillions of messages                                │
│  Original: Cassandra → Migrated to ScyllaDB (faster C++)    │
│                                                              │
│  Partition Key: (channel_id, bucket)                         │
│                                                              │
│  bucket = time window (e.g., 10-day intervals)              │
│                                                              │
│  Why bucket?                                                 │
│  → Without it: One hot channel = one MASSIVE partition       │
│  → With bucket: Data spread across time buckets              │
│  → #general with 5M messages → split into 500 partitions    │
│                                                              │
│  The "large message" problem:                                │
│  → One server had a channel with 1M+ messages in a bucket   │
│  → Reads took 40+ seconds → timeout → retry storm           │
│  → Solution: Smaller time buckets for active channels        │
└─────────────────────────────────────────────────────────────┘
```

### Notion (PostgreSQL Sharding)

```
┌─────────────────────────────────────────────────────────────┐
│  NOTION'S APPROACH (2021 Migration):                         │
│                                                              │
│  Before: Single PostgreSQL instance (480 GB)                │
│  Problem: 5-minute table lock during migration = outage     │
│                                                              │
│  Solution: Double-write migration                            │
│                                                              │
│  Phase 1: Write to OLD db + NEW sharded db simultaneously   │
│  Phase 2: Backfill historical data to new shards            │
│  Phase 3: Verify data consistency between old and new       │
│  Phase 4: Switch reads to new sharded db                    │
│  Phase 5: Stop writes to old db                             │
│                                                              │
│  Shard Key: workspace_id (team/company)                     │
│  → All of a team's pages, blocks, comments = one shard      │
│  → 480 logical shards → mapped to 32 physical databases     │
│                                                              │
│  Result: 💀 → 😌                                            │
│  → P99 latency dropped from 10s to 50ms                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 🚀 Resharding — The Hardest Operation in Databases

### Why Resharding Happens

```
Scenario: Your 4 shards are now unbalanced

  Shard 1: 200 GB  (25%) 😌
  Shard 2: 800 GB  (60%) 🔥🔥🔥  ← Celebrity user signed up here
  Shard 3: 150 GB  (10%) 😌
  Shard 4:  50 GB  ( 5%) 😌

  You need to split Shard 2 or redistribute data.
```

### Resharding Strategies

```
┌──────────────────────────────────────────────────────────────┐
│  STRATEGY 1: LOGICAL SHARDS (Best Practice)                   │
│                                                               │
│  Instead of N physical shards, create N × 100 LOGICAL shards │
│  Map logical shards → physical servers                        │
│                                                               │
│  Physical Server 1:  logical_shard_001 to logical_shard_050  │
│  Physical Server 2:  logical_shard_051 to logical_shard_100  │
│                                                               │
│  To reshards: just remap logical → physical!                  │
│  Move logical_shard_025 from Server 1 → Server 3             │
│  No rehashing needed! ✅                                      │
├──────────────────────────────────────────────────────────────┤
│  STRATEGY 2: DOUBLE-WRITE MIGRATION                           │
│                                                               │
│  Phase 1: Write to old shards + new shards simultaneously    │
│  Phase 2: Backfill old data to new shard layout              │
│  Phase 3: Verify data consistency                             │
│  Phase 4: Switch reads to new layout                          │
│  Phase 5: Stop writes to old layout                           │
│                                                               │
│  Used by: Notion, Stripe, GitHub                              │
├──────────────────────────────────────────────────────────────┤
│  STRATEGY 3: GHOST TABLE (for MySQL via gh-ost)               │
│                                                               │
│  1. Create new table with new sharding layout                 │
│  2. Copy data row-by-row in background                        │
│  3. Capture changes via binlog during copy                    │
│  4. Atomic table swap when done                               │
│                                                               │
│  Zero downtime! Used by: GitHub, Shopify                      │
└──────────────────────────────────────────────────────────────┘
```

---

## ❌ Top 10 Sharding Mistakes

```
┌──────────────────────────────────────────────────────────────────┐
│                  🚨 TOP 10 SHARDING MISTAKES                     │
│                                                                   │
│  1. Sharding too early                                            │
│     → Try everything else first (indexes, caching, replicas)     │
│                                                                   │
│  2. Choosing a bad shard key                                      │
│     → Low cardinality, monotonic, mutable = pain forever          │
│                                                                   │
│  3. Not planning for cross-shard queries                          │
│     → "We'll figure it out later" = regret                       │
│                                                                   │
│  4. Forgetting about cross-shard transactions                     │
│     → Your money transfer feature? It's broken now.               │
│                                                                   │
│  5. Ignoring data locality                                        │
│     → Related data on different shards = slow JOINs               │
│                                                                   │
│  6. Not using logical shards                                      │
│     → Directly mapping to physical servers = resharding hell      │
│                                                                   │
│  7. Unbalanced shard keys                                         │
│     → "US" shard has 10x more data than "Antarctica" shard       │
│                                                                   │
│  8. No monitoring per shard                                       │
│     → One hot shard can bring down everything                     │
│                                                                   │
│  9. Not replicating reference/lookup tables                       │
│     → Every JOIN requires cross-shard hop                         │
│                                                                   │
│  10. Using auto-increment IDs as shard keys                       │
│      → All new inserts → last shard = permanent hotspot           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🧪 Sharding Decision Framework

```
                Should You Shard?
                ═════════════════

                ┌──────────────────┐
                │ Is your database │
                │ actually slow?   │
                └────────┬─────────┘
                         │
                    ┌────▼────┐
                    │   NO    │──► Don't shard. Go home. 🏠
                    └─────────┘
                    ┌────▼────┐
                    │   YES   │
                    └────┬────┘
                         │
                ┌────────▼────────┐
                │ Have you tried  │
                │ query           │──► NO ──► Do that first!
                │ optimization?   │
                └────────┬────────┘
                         │ YES
                ┌────────▼────────┐
                │ Have you tried  │
                │ read replicas?  │──► NO ──► Do that first!
                └────────┬────────┘
                         │ YES
                ┌────────▼────────┐
                │ Have you tried  │
                │ caching?        │──► NO ──► Do that first!
                └────────┬────────┘
                         │ YES
                ┌────────▼────────┐
                │ Have you tried  │
                │ vertical scaling│──► NO ──► Do that first!
                └────────┬────────┘
                         │ YES
                ┌────────▼────────┐
                │ Is your WRITE   │
                │ throughput the  │──► NO ──► Maybe partition,
                │ bottleneck?     │         not shard
                └────────┬────────┘
                         │ YES
                ┌────────▼────────┐
                │ Is your data    │
                │ > 1 TB?         │──► NO ──► Try partitioning first
                └────────┬────────┘
                         │ YES
                         │
                    ┌────▼────┐
                    │  SHARD  │ 🔪
                    │  TIME!  │
                    └─────────┘
```

---

## 📊 Sharding vs Partitioning — Know the Difference!

```
┌───────────────────┬───────────────────────┬──────────────────────┐
│                   │     PARTITIONING      │      SHARDING        │
├───────────────────┼───────────────────────┼──────────────────────┤
│ Data location     │ Same server, split    │ Different servers     │
│                   │ into segments          │ entirely             │
│                   │                        │                      │
│ Who manages it?   │ Database engine        │ Application / proxy  │
│                   │ (built-in feature)     │ / middleware          │
│                   │                        │                      │
│ Transparency      │ Transparent to app     │ App must be shard-   │
│                   │                        │ aware (usually)      │
│                   │                        │                      │
│ Transactions      │ Normal (same server)   │ Distributed (hard!)  │
│                   │                        │                      │
│ JOINs             │ Normal (same server)   │ Cross-shard (hard!)  │
│                   │                        │                      │
│ Write scaling     │ ❌ Still one server    │ ✅ Multiple servers  │
│                   │                        │                      │
│ Use case          │ Large tables on one    │ Data too big for     │
│                   │ server                  │ any single server    │
│                   │                        │                      │
│ Examples          │ PostgreSQL PARTITION   │ MongoDB shards,      │
│                   │ Oracle Partitioning     │ Citus, Vitess        │
└───────────────────┴───────────────────────┴──────────────────────┘

💡 Think of it this way:
   Partitioning = Organizing files into folders on ONE computer
   Sharding     = Spreading files across MULTIPLE computers
```

---

## 🎯 Quick Reference — Sharding Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    SHARDING CHEAT SHEET                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Shard Key Selection:                                             │
│  ✅ High cardinality        ❌ Low cardinality                   │
│  ✅ Even distribution       ❌ Skewed distribution               │
│  ✅ Query-aligned           ❌ Random / unrelated to queries     │
│  ✅ Immutable               ❌ Frequently changing               │
│  ✅ Non-monotonic           ❌ Auto-increment / timestamp only   │
│                                                                   │
│  Strategy Selection:                                              │
│  Hash  → When you need even distribution + point lookups         │
│  Range → When you need range scans + time-series data            │
│  Dir   → When you need max flexibility + multi-tenant            │
│                                                                   │
│  Cross-Shard Solutions:                                           │
│  Queries → Denormalize / CQRS / Search index                    │
│  JOINs   → Reference tables / Denormalize / App-level join      │
│  Txns    → 2PC / Saga / Co-locate data                          │
│  IDs     → Snowflake / ULID / UUID                              │
│                                                                   │
│  Golden Rules:                                                    │
│  1. Shard as late as possible                                     │
│  2. Use logical shards (not physical) from day one               │
│  3. Monitor per-shard metrics                                     │
│  4. Plan for resharding before you need it                        │
│  5. Replicate small reference tables to all shards                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

Now that you understand how to **split data**, the next chapter covers **Replication Strategies Deep Dive** — how to **copy data** across nodes for high availability and read scaling.

> **Chapter 7.2:** [Replication Strategies Deep Dive →](./02-Replication.md)

---

> _"Sharding is easy to describe, hard to implement, and a nightmare to undo. Make sure you really need it before you start."_
> — Every senior database engineer, ever
