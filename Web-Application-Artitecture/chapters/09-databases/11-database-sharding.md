# Database Sharding — Splitting Data Across Machines

> **What you'll learn**: How to distribute data across multiple database servers when one server can't handle the load, different sharding strategies (hash, range, directory), the challenges of cross-shard queries, and how companies like Instagram and Uber shard at planet scale.

---

## Real-Life Analogy

Imagine a **library** that has grown so large that one building can't hold all the books. You decide to build **multiple buildings**:

- **Building A**: Books by authors whose last names start with A-F
- **Building B**: Authors G-M
- **Building C**: Authors N-S
- **Building D**: Authors T-Z

Now each building handles 25% of the visitors. But there's a catch: if someone asks "Show me all books published in 2024 across ALL authors," you need to ask ALL four buildings and combine the answers. That's the fundamental trade-off of sharding.

---

## Core Concept Explained Step-by-Step

### What is Sharding?

**Sharding** = splitting one large database table across multiple database servers (shards), each holding a portion of the data.

```
BEFORE SHARDING (Single Server):              AFTER SHARDING (4 Servers):
┌─────────────────────────────┐             ┌────────────┐ ┌────────────┐
│   ALL 1 BILLION USERS       │             │  Shard 1   │ │  Shard 2   │
│   on ONE server             │             │ Users 1-   │ │ Users      │
│                             │             │ 250M       │ │ 250M-500M  │
│   Problems:                 │             └────────────┘ └────────────┘
│   • Disk full (10TB+)      │             ┌────────────┐ ┌────────────┐
│   • Queries slow            │             │  Shard 3   │ │  Shard 4   │
│   • Writes bottlenecked    │             │ Users      │ │ Users      │
│   • Single point of failure│             │ 500M-750M  │ │ 750M-1B   │
└─────────────────────────────┘             └────────────┘ └────────────┘
                                            
                                            Each shard: ~250M users
                                            Each shard: independent server
                                            Write capacity: 4x!
```

### Sharding vs Replication

```
┌──────────────────────────────────────────────────────────────────┐
│              REPLICATION vs SHARDING                               │
├─────────────────────────────┬────────────────────────────────────┤
│      REPLICATION             │          SHARDING                  │
├─────────────────────────────┼────────────────────────────────────┤
│ Same data on all servers    │ Different data on each server      │
│ Scales READS               │ Scales READS and WRITES            │
│ Every server has full copy  │ Each server has a PORTION          │
│ Easy (built into all DBs)  │ Complex (application logic needed) │
│ High availability          │ High capacity                      │
└─────────────────────────────┴────────────────────────────────────┘

COMBINED (Real-world):
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Shard 1    │   │   Shard 2    │   │   Shard 3    │
│   Primary    │   │   Primary    │   │   Primary    │
│   + 2 Replicas│  │   + 2 Replicas│  │   + 2 Replicas│
│   Users A-H  │   │   Users I-P  │   │   Users Q-Z  │
└──────────────┘   └──────────────┘   └──────────────┘

Each shard is replicated for availability
Data is split for write scalability
```

### Sharding Strategies

```
1. HASH-BASED SHARDING:
────────────────────────────────────────
shard_number = hash(shard_key) % number_of_shards

user_id = 12345
hash(12345) = 8274619
8274619 % 4 = 3  → Goes to Shard 3

✅ Even distribution (no hot spots)
✅ Simple to implement
❌ Range queries across shards (inefficient)
❌ Adding shards requires resharding (redistribute data)

2. RANGE-BASED SHARDING:
────────────────────────────────────────
Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000

✅ Range queries efficient (within one shard)
✅ Easy to understand
❌ Hot spots (new users all go to latest shard)
❌ Uneven distribution over time

3. DIRECTORY-BASED SHARDING:
────────────────────────────────────────
Lookup table maps each key to its shard:
  user_123 → Shard 2
  user_456 → Shard 1
  user_789 → Shard 3

✅ Flexible (can move data between shards)
✅ Handles uneven distribution
❌ Lookup table is a single point of failure
❌ Extra network hop for every query

4. GEOGRAPHIC SHARDING:
────────────────────────────────────────
Shard by user's region:
  India users    → Mumbai shard
  US users       → Virginia shard
  Europe users   → Frankfurt shard

✅ Low latency (data near users)
✅ Data sovereignty compliance
❌ Uneven shard sizes (India shard may be huge)
❌ Users who travel see different data
```

### Choosing a Shard Key

```
GOOD SHARD KEYS:                         BAD SHARD KEYS:
───────────────────────────────          ───────────────────────────────
user_id                                  timestamp (hot spot: all writes
  → Even distribution                     go to current time shard)
  → Most queries include user_id        
                                         country (uneven: India/US = huge,
tenant_id (multi-tenant SaaS)             Luxembourg = tiny)
  → Data isolation per tenant           
  → Queries scoped to tenant            status (only 3 values: active,
                                           inactive, banned → 3 shards max)
order_id (with hash)                    
  → Even distribution                   auto_increment_id with range
  → Each order on one shard               sharding (newest shard = hot)
```

---

## How It Works Internally

### Application-Level Sharding

```
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   SHARD ROUTER                            │   │
│  │                                                          │   │
│  │  Input: user_id = 12345                                  │   │
│  │  Step 1: shard_num = hash(12345) % 4 = 3                │   │
│  │  Step 2: lookup shard_config[3] → "db-shard-3.internal" │   │
│  │  Step 3: route query to that connection pool             │   │
│  └──────────┬──────────────┬──────────────┬─────────────────┘   │
│             │              │              │                       │
│             ▼              ▼              ▼                       │
│     ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│     │  Shard 0   │  │  Shard 1   │  │  Shard 2   │  ...       │
│     │  Pool      │  │  Pool      │  │  Pool      │             │
│     └────────────┘  └────────────┘  └────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### Proxy-Based Sharding

```
┌─────────┐     ┌────────────────────┐     ┌──────────────┐
│  App    │────▶│  Shard Proxy       │────▶│  Shard 0     │
│ Server  │     │  (Vitess/ProxySQL/ │     │  (MySQL)     │
│         │     │   Citus)           │────▶│  Shard 1     │
└─────────┘     │                    │────▶│  Shard 2     │
                │  • Parses query    │     └──────────────┘
                │  • Determines shard│
                │  • Routes request  │
                │  • Merges results  │
                │    (scatter-gather)│
                └────────────────────┘
```

### Cross-Shard Queries (The Hard Part)

```
Query: "SELECT * FROM users WHERE age > 25 ORDER BY name LIMIT 10"

Problem: age and name aren't the shard key (user_id is)!
Data matching this query is scattered across ALL shards!

SCATTER-GATHER Pattern:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  Coordinator sends query to ALL shards:                      │
│                                                               │
│  Shard 0: "SELECT * FROM users WHERE age>25 ORDER BY name   │
│            LIMIT 10"  → returns 10 results                   │
│  Shard 1: same query → returns 10 results                   │
│  Shard 2: same query → returns 10 results                   │
│  Shard 3: same query → returns 10 results                   │
│                                                               │
│  Coordinator: Merge 40 results, sort by name, take top 10   │
│                                                               │
│  ⚠️ Expensive: queries ALL shards even though answer might   │
│     be on just one shard!                                    │
└──────────────────────────────────────────────────────────────┘
```

### Resharding — Adding More Shards

```
BEFORE (4 shards):     hash(key) % 4
AFTER (8 shards):      hash(key) % 8

Problem: EVERY key gets a different shard number!
  hash(12345) % 4 = 3  →  hash(12345) % 8 = 7 (MOVED!)
  
  Nearly ALL data must be migrated! 😱

SOLUTION: Consistent Hashing
  Only ~1/N of keys move when adding one shard
  (Covered in Chapter 13.10: Consistent Hashing)

SOLUTION: Virtual shards (over-provision)
  Create 1024 virtual shards from day one
  Map virtual shards to physical servers:
    Virtual 0-255   → Physical Server A
    Virtual 256-511 → Physical Server B
    Virtual 512-767 → Physical Server C
    Virtual 768-1023 → Physical Server D
  
  To add a server: just reassign some virtual shards
  Only those virtual shards' data migrates!
```

---

## Code Examples

### Python (Application-Level Sharding)

```python
import hashlib
from typing import Dict, Any
import psycopg2
from psycopg2.pool import ThreadedConnectionPool

class ShardRouter:
    """Routes queries to the correct database shard"""
    
    def __init__(self, shard_configs: list):
        self.num_shards = len(shard_configs)
        self.pools: Dict[int, ThreadedConnectionPool] = {}
        
        for i, config in enumerate(shard_configs):
            self.pools[i] = ThreadedConnectionPool(
                minconn=5, maxconn=20,
                host=config['host'], port=config['port'],
                dbname=config['dbname'], 
                user=config['user'], password=config['password']
            )
    
    def get_shard(self, shard_key: int) -> int:
        """Determine which shard holds this key"""
        # Consistent hashing would be better for production
        hash_val = int(hashlib.md5(str(shard_key).encode()).hexdigest(), 16)
        return hash_val % self.num_shards
    
    def get_connection(self, shard_key: int):
        """Get a connection to the correct shard"""
        shard_id = self.get_shard(shard_key)
        return self.pools[shard_id].getconn(), shard_id
    
    def execute_on_shard(self, shard_key: int, sql: str, params=None):
        """Execute query on the shard that holds this key"""
        conn, shard_id = self.get_connection(shard_key)
        try:
            cursor = conn.cursor()
            cursor.execute(sql, params)
            conn.commit()
            return cursor.fetchall() if cursor.description else None
        finally:
            self.pools[shard_id].putconn(conn)
    
    def execute_on_all_shards(self, sql: str, params=None):
        """Scatter-gather: query all shards and merge results"""
        all_results = []
        for shard_id in range(self.num_shards):
            conn = self.pools[shard_id].getconn()
            try:
                cursor = conn.cursor()
                cursor.execute(sql, params)
                results = cursor.fetchall()
                all_results.extend(results)
            finally:
                self.pools[shard_id].putconn(conn)
        return all_results

# Configuration
shard_configs = [
    {"host": "shard0.db.internal", "port": 5432, "dbname": "users", "user": "app", "password": "s3cret"},
    {"host": "shard1.db.internal", "port": 5432, "dbname": "users", "user": "app", "password": "s3cret"},
    {"host": "shard2.db.internal", "port": 5432, "dbname": "users", "user": "app", "password": "s3cret"},
    {"host": "shard3.db.internal", "port": 5432, "dbname": "users", "user": "app", "password": "s3cret"},
]

router = ShardRouter(shard_configs)

# Write: goes to correct shard based on user_id
user_id = 12345
router.execute_on_shard(
    shard_key=user_id,
    sql="INSERT INTO users (id, name, email) VALUES (%s, %s, %s)",
    params=(user_id, "Alice", "alice@example.com")
)

# Read by shard key: single shard (fast!)
user = router.execute_on_shard(
    shard_key=user_id,
    sql="SELECT * FROM users WHERE id = %s",
    params=(user_id,)
)

# Cross-shard query (slow — hits ALL shards)
all_active = router.execute_on_all_shards(
    sql="SELECT * FROM users WHERE status = 'active' ORDER BY created_at DESC LIMIT 10"
)
# Must sort merged results in application!
all_active.sort(key=lambda x: x[4], reverse=True)  # Sort by created_at
```

### Java (Sharding with ShardingSphere)

```java
import org.apache.shardingsphere.api.sharding.standard.PreciseShardingAlgorithm;
import org.apache.shardingsphere.api.sharding.standard.PreciseShardingValue;
import java.util.Collection;

// Custom sharding algorithm
public class UserIdShardingAlgorithm implements PreciseShardingAlgorithm<Long> {
    
    @Override
    public String doSharding(Collection<String> availableTargetNames,
                             PreciseShardingValue<Long> shardingValue) {
        Long userId = shardingValue.getValue();
        int shardIndex = (int) (userId % availableTargetNames.size());
        
        // Return the datasource name like "ds_0", "ds_1", etc.
        return "ds_" + shardIndex;
    }
}

// Using JPA with sharding (transparent to business logic)
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public User createUser(User user) {
        // ShardingSphere automatically routes to correct shard
        // based on user.getId() (shard key)
        return userRepository.save(user);
    }

    public User findById(Long userId) {
        // Single-shard query (fast)
        return userRepository.findById(userId).orElseThrow();
    }

    public List<User> findByCity(String city) {
        // Cross-shard query (scatter-gather, slower)
        // ShardingSphere handles merging results from all shards
        return userRepository.findByCity(city);
    }
}
```

---

## Infrastructure Examples

### Vitess (YouTube's Sharding Solution) - Kubernetes

```yaml
# Vitess cluster on Kubernetes
apiVersion: planetscale.com/v2
kind: VitessCluster
metadata:
  name: myapp
spec:
  cells:
    - name: zone1
      gateway:
        replicas: 2
  keyspaces:
    - name: users
      turndownPolicy: Immediate
      partitionings:
        - equal:
            parts: 4  # 4 shards
            shardTemplate:
              databaseInitScriptSecret:
                name: myapp-init-script
              tabletPools:
                - cell: zone1
                  type: replica
                  replicas: 3
                  mysqld:
                    resources:
                      requests:
                        cpu: "2"
                        memory: "4Gi"
                  dataVolumeClaimTemplate:
                    accessModes: ["ReadWriteOnce"]
                    resources:
                      requests:
                        storage: 100Gi
```

### Citus (PostgreSQL Sharding Extension)

```sql
-- Citus: Distributed PostgreSQL (sharding built-in)

-- Create distributed table (sharded by user_id)
SELECT create_distributed_table('orders', 'user_id');

-- Queries work as normal SQL!
-- Single-shard (fast): query includes shard key
SELECT * FROM orders WHERE user_id = 12345;

-- Cross-shard (automatic scatter-gather):
SELECT status, COUNT(*), SUM(amount)
FROM orders
GROUP BY status;

-- Co-located tables: shard by same key for efficient joins
SELECT create_distributed_table('order_items', 'user_id');

-- This JOIN is LOCAL (same shard, same user_id):
SELECT o.id, oi.product_name
FROM orders o
JOIN order_items oi ON o.id = oi.order_id AND o.user_id = oi.user_id
WHERE o.user_id = 12345;
```

---

## Real-World Example

### Instagram — Sharding PostgreSQL

```
┌─────────────────────────────────────────────────────────────────┐
│            Instagram's Sharding Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Logical Shards: 4096 (fixed from the start)                    │
│  Physical Servers: ~64 (each holds 64 logical shards)           │
│                                                                   │
│  Shard Key: user_id                                              │
│  Shard Function: user_id % 4096 → logical_shard_id              │
│  Mapping: logical_shard_id → physical_server                     │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ ID Generation (guarantees IDs encode the shard!)       │     │
│  │                                                        │     │
│  │ 64-bit ID structure:                                   │     │
│  │ ┌──────────────┬───────────────┬────────────────────┐ │     │
│  │ │ 41 bits      │ 13 bits       │ 10 bits            │ │     │
│  │ │ timestamp(ms)│ shard_id      │ sequence_in_shard  │ │     │
│  │ └──────────────┴───────────────┴────────────────────┘ │     │
│  │                                                        │     │
│  │ From any ID, you can extract which shard it's on!     │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                   │
│  Growing: Move logical shards to new physical servers            │
│  (Only 1/64 of data moves when adding one server)               │
│                                                                   │
│  Scale: 2B+ users, 64 physical PostgreSQL clusters              │
│  Each cluster: Primary + 3 Replicas + PgBouncer                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Sharding too early | Massive complexity for no benefit | Only shard when single server is maxed out |
| Wrong shard key | Hot spots or too many cross-shard queries | Choose key present in 80%+ of queries |
| Not planning for resharding | Data grows, shards become unbalanced | Use virtual shards from day one (over-provision) |
| Cross-shard JOINs | N-shard scatter-gather is slow | Denormalize or co-locate related data |
| Cross-shard transactions | 2PC is slow and complex | Use Saga pattern or eventual consistency |
| Forgetting about migrations | Schema changes on 64 shards simultaneously | Use tools like gh-ost, pt-online-schema-change |
| Global unique IDs | auto_increment conflicts across shards | Use Snowflake IDs, UUIDs, or shard-encoded IDs |
| Uneven shard sizes | One shard 10x larger than others | Implement rebalancing / split large shards |

---

## When to Use / When NOT to Use

### ✅ Use Sharding When:
- Single database server is **at capacity** (CPU, disk, connections)
- Data exceeds **what one server can store** (multi-TB)
- **Write throughput** exceeds single-server capacity
- Need to **isolate tenants** (each tenant = own shard)
- Regulatory requirements demand **data locality** (geo-sharding)
- Already tried: read replicas, caching, query optimization, vertical scaling

### ❌ Do NOT Shard When:
- Data fits on one server (< 500GB for most workloads)
- **Read scaling** is the issue (use replicas instead)
- You haven't optimized **queries and indexes** first
- Your team doesn't have the **operational expertise**
- Most queries are **cross-shard** (defeating the purpose)
- A managed service handles it (use Aurora, Spanner, CockroachDB)

---

## Key Takeaways

1. **Sharding splits data across multiple servers** — each server holds a portion. It's the only way to scale writes beyond one server's capacity.
2. **Choose shard key carefully** — it determines data distribution AND which queries are efficient. Most queries should include the shard key.
3. **Hash-based sharding** gives even distribution; **range-based** enables range queries but creates hot spots.
4. **Cross-shard queries are expensive** — they require scatter-gather across all shards. Design your data model to minimize them.
5. **Virtual shards** (over-provisioning logical shards) make future resharding much easier — create 1024+ logical shards even if you start with 4 physical servers.
6. **Sharding is a last resort** — try vertical scaling, read replicas, caching, and query optimization first. Sharding adds massive operational complexity.
7. **Tools like Vitess, Citus, and ShardingSphere** can abstract sharding complexity, but you still need to understand the trade-offs.

---

## What's Next?

Next, we'll explore **Database Partitioning (Horizontal & Vertical)** (Chapter 9.12), which is often confused with sharding but is actually a different technique that splits data within a single database server.
