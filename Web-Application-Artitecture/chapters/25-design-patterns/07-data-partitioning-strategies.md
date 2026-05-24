# Data Partitioning Strategies for Planet-Scale Apps

> **What you'll learn**: How companies like Google, Amazon, and Facebook split trillions of rows across thousands of machines — covering horizontal/vertical partitioning, sharding strategies, consistent hashing, hot spots, rebalancing, and the trade-offs that define planet-scale data architecture.

---

## Real-Life Analogy

Imagine you're running a library with **1 billion books**. One building can't hold them all. So you:

1. **Split by subject** (Science in Building A, History in Building B) — that's **vertical partitioning**
2. **Split by author's last name** (A-M in Building A, N-Z in Building B) — that's **horizontal partitioning / sharding**
3. **Split by geography** (Delhi branch has Hindi books, London has English) — that's **geo-partitioning**

Each approach has trade-offs:
- Splitting by subject makes cross-subject research hard
- Splitting by author name means popular authors (A. for "Anonymous") fill up one building
- Splitting by geography means you can't easily get a Delhi book in London

Data partitioning is the same problem — but with databases at internet scale.

---

## Why Partition Data?

```
Single Database Problem:
────────────────────────────────────────────────
• 10 TB of data → Doesn't fit on one disk
• 100,000 queries/sec → One server can't handle it
• One region → Users 10,000 km away get 200ms latency
• One machine → Single point of failure

Solution: Split data across multiple machines
────────────────────────────────────────────────
• 10 TB ÷ 100 machines = 100 GB each ✅
• 100K qps ÷ 100 machines = 1K qps each ✅
• Data in multiple regions → Low latency ✅
• Redundancy → No single point of failure ✅
```

---

## Types of Data Partitioning

```
┌─────────────────────────────────────────────────────────────────┐
│              DATA PARTITIONING TAXONOMY                           │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           VERTICAL PARTITIONING                           │    │
│  │  Split a table into multiple tables by COLUMNS            │    │
│  │                                                           │    │
│  │  users_core: id, name, email                              │    │
│  │  users_profile: id, bio, avatar, preferences              │    │
│  │  users_activity: id, last_login, sessions                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           HORIZONTAL PARTITIONING (SHARDING)              │    │
│  │  Split a table into multiple tables by ROWS               │    │
│  │                                                           │    │
│  │  Shard 1: users where id 1 - 1,000,000                   │    │
│  │  Shard 2: users where id 1,000,001 - 2,000,000           │    │
│  │  Shard 3: users where id 2,000,001 - 3,000,000           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           FUNCTIONAL / DOMAIN PARTITIONING                │    │
│  │  Split by business function/domain                        │    │
│  │                                                           │    │
│  │  Orders DB → handles all order data                       │    │
│  │  Users DB → handles all user data                         │    │
│  │  Products DB → handles all product data                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Sharding Strategies (Horizontal Partitioning)

### Strategy 1: Range-Based Sharding

Assign rows to shards based on a range of the shard key.

```
┌─────────────────────────────────────────────────────────────┐
│               RANGE-BASED SHARDING                           │
│                                                               │
│  Shard Key: user_id                                          │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │ Shard 4  │   │
│  │          │  │          │  │          │  │          │   │
│  │ IDs:     │  │ IDs:     │  │ IDs:     │  │ IDs:     │   │
│  │ 1 - 1M   │  │ 1M - 2M  │  │ 2M - 3M  │  │ 3M - 4M  │   │
│  │          │  │          │  │          │  │          │   │
│  │ Server A │  │ Server B │  │ Server C │  │ Server D │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                               │
│  Query: SELECT * FROM users WHERE user_id = 2,500,000        │
│         → Route to Shard 3 (range 2M-3M)                    │
│                                                               │
│  ✅ Pros: Range queries efficient, simple to understand      │
│  ❌ Cons: Hot spots (new users all hit Shard 4),             │
│           uneven distribution                                │
└─────────────────────────────────────────────────────────────┘
```

### Strategy 2: Hash-Based Sharding

Apply a hash function to the shard key and use modulo to pick the shard.

```
┌─────────────────────────────────────────────────────────────┐
│               HASH-BASED SHARDING                            │
│                                                               │
│  Shard = hash(user_id) % num_shards                          │
│                                                               │
│  user_id = 12345                                             │
│  hash(12345) = 987654321                                     │
│  987654321 % 4 = 1  → Shard 1                               │
│                                                               │
│  user_id = 67890                                             │
│  hash(67890) = 123456789                                     │
│  123456789 % 4 = 1  → Shard 1                               │
│                                                               │
│  user_id = 11111                                             │
│  hash(11111) = 555555555                                     │
│  555555555 % 4 = 3  → Shard 3                               │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │   │
│  │ ~25% data│  │ ~25% data│  │ ~25% data│  │ ~25% data│   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                               │
│  ✅ Pros: Even distribution, no hot spots                    │
│  ❌ Cons: Range queries IMPOSSIBLE (must query all shards),  │
│           adding/removing shards requires reshuffling         │
└─────────────────────────────────────────────────────────────┘
```

### Strategy 3: Consistent Hashing (The Industry Standard)

Solves the "adding/removing shards" problem of simple hash-based sharding.

```
┌─────────────────────────────────────────────────────────────┐
│            CONSISTENT HASHING RING                            │
│                                                               │
│                    0° (top)                                    │
│                      │                                        │
│              ┌───────┼───────┐                                │
│             /        │        \                               │
│           /    Node A │   Node B \                            │
│          │    (0°-90°)│(90°-180°) │                           │
│          │           │            │                           │
│  270°────┤           │            ├────90°                    │
│          │           │            │                           │
│          │  Node D   │   Node C   │                           │
│           \ (270°-0°)│(180°-270°)/                           │
│             \        │        /                               │
│              └───────┼───────┘                                │
│                      │                                        │
│                   180°                                         │
│                                                               │
│  Key "user_123" hashes to 45° → lands on Node A              │
│  Key "order_456" hashes to 200° → lands on Node C            │
│                                                               │
│  Add Node E at 135°:                                          │
│    Only keys between 90°-135° move (from Node B to Node E)   │
│    Everything else stays! (Minimal data movement)             │
│                                                               │
│  ✅ Adding/removing nodes moves only ~1/N of the data        │
│  ✅ Used by: DynamoDB, Cassandra, Riak, Memcached            │
└─────────────────────────────────────────────────────────────┘
```

### Strategy 4: Directory-Based Sharding (Lookup Table)

A central directory maps each key to its shard location.

```
┌──────────────────────────────────────────┐
│          DIRECTORY SERVICE                 │
│                                            │
│  ┌──────────────────────────────────┐    │
│  │  user_id  │  shard_location       │    │
│  ├───────────┼───────────────────────┤    │
│  │  user_1   │  shard-3.region-us    │    │
│  │  user_2   │  shard-1.region-eu    │    │
│  │  user_3   │  shard-2.region-us    │    │
│  │  user_4   │  shard-1.region-ap    │    │
│  └───────────┴───────────────────────┘    │
└──────────────────────────────────────────┘
          │
    Query: Where is user_3?
    Answer: shard-2.region-us
          │
          ▼
┌──────────────────────────────────────────┐
│  shard-2.region-us                        │
│  (Has user_3's data)                      │
└──────────────────────────────────────────┘

✅ Pros: Maximum flexibility, can move users between shards
❌ Cons: Directory is a bottleneck + single point of failure
         Must be cached, replicated, and highly available
```

### Strategy 5: Geo-Based Sharding

Partition data by geographic location.

```
┌─────────────────────────────────────────────────────────────┐
│               GEO-BASED SHARDING                             │
│                                                               │
│          🌍 Global Traffic                                    │
│               │                                               │
│     ┌─────────┼─────────┐                                    │
│     ▼         ▼         ▼                                    │
│  ┌────────┐ ┌────────┐ ┌────────┐                           │
│  │US Shard│ │EU Shard│ │AP Shard│                           │
│  │        │ │        │ │        │                           │
│  │ Users: │ │ Users: │ │ Users: │                           │
│  │ US, CA │ │ UK, DE │ │ IN, JP │                           │
│  │ MX, BR │ │ FR, IT │ │ AU, SG │                           │
│  │        │ │        │ │        │                           │
│  │us-east │ │eu-west │ │ap-south│                           │
│  └────────┘ └────────┘ └────────┘                           │
│                                                               │
│  ✅ Pros: Low latency, data residency compliance             │
│  ❌ Cons: Cross-region queries are expensive/slow            │
│           User traveling abroad → higher latency             │
│                                                               │
│  Used by: Google Spanner, CockroachDB, DynamoDB Global       │
└─────────────────────────────────────────────────────────────┘
```

---

## Choosing a Shard Key — The Most Critical Decision

```
┌─────────────────────────────────────────────────────────────┐
│           SHARD KEY SELECTION CRITERIA                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  A good shard key must have:                                 │
│                                                               │
│  1. HIGH CARDINALITY                                         │
│     Many distinct values → even distribution                 │
│     ✅ user_id (millions of values)                          │
│     ❌ country (only ~200 values → uneven)                   │
│     ❌ boolean status (only 2 values!)                        │
│                                                               │
│  2. EVEN DISTRIBUTION                                        │
│     Data spread equally across shards                        │
│     ✅ UUID/hash of user_id → uniform                        │
│     ❌ timestamp → all new data hits one shard               │
│     ❌ celebrity user_id → one shard gets all followers       │
│                                                               │
│  3. QUERY ALIGNMENT                                          │
│     Most queries include the shard key                       │
│     ✅ user_id (most queries are "get data for user X")      │
│     ❌ random_field (every query is a scatter-gather)         │
│                                                               │
│  4. STABILITY                                                │
│     Value doesn't change over time                           │
│     ✅ user_id (never changes)                               │
│     ❌ user_location (changes when user moves → resharding)  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally — Request Routing

```
Client sends: GET /users/12345
                    │
                    ▼
┌───────────────────────────────────────────────────────────┐
│              ROUTING LAYER                                  │
│                                                             │
│  Option A: Application-level routing                       │
│  ┌─────────────────────────────────────────────────┐     │
│  │  shard = hash(12345) % num_shards               │     │
│  │  shard = 987654321 % 4 = 1                      │     │
│  │  → Connect to shard-1 database                   │     │
│  └─────────────────────────────────────────────────┘     │
│                                                             │
│  Option B: Proxy-level routing (Vitess, ProxySQL)         │
│  ┌─────────────────────────────────────────────────┐     │
│  │  App → Proxy → Proxy determines shard → DB       │     │
│  │  App doesn't know about sharding!                │     │
│  └─────────────────────────────────────────────────┘     │
│                                                             │
│  Option C: Client library routing (Cassandra driver)      │
│  ┌─────────────────────────────────────────────────┐     │
│  │  Smart client knows cluster topology             │     │
│  │  Routes directly to correct node                 │     │
│  └─────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────┘
```

---

## Handling Cross-Shard Queries (The Hard Part)

```
EASY: Query with shard key
─────────────────────────────────────────
SELECT * FROM orders WHERE user_id = 123
→ Route to one shard. Fast!

HARD: Query without shard key (scatter-gather)
─────────────────────────────────────────
SELECT * FROM orders WHERE total > 1000 ORDER BY created_at LIMIT 10

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │ Shard 4  │
│          │  │          │  │          │  │          │
│ Query    │  │ Query    │  │ Query    │  │ Query    │
│ locally  │  │ locally  │  │ locally  │  │ locally  │
└─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘
      │              │              │              │
      └──────────────┴──────────────┴──────────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Coordinator   │
                  │  Merge results │
                  │  Sort + Limit  │
                  └───────────────┘
                          │
                          ▼
                    Final 10 results

VERY HARD: Cross-shard JOIN
─────────────────────────────────────────
SELECT * FROM orders o 
JOIN users u ON o.user_id = u.id
WHERE u.country = 'India'

This is why sharding should be AVOIDED until necessary!
```

---

## Code Examples

### Python: Application-Level Sharding

```python
import hashlib
from typing import Dict, Any
import psycopg2

class ShardRouter:
    """Routes queries to the correct shard based on shard key"""
    
    def __init__(self, shard_configs: list):
        """
        shard_configs: [
            {"host": "shard-0.db", "port": 5432, "dbname": "app"},
            {"host": "shard-1.db", "port": 5432, "dbname": "app"},
            {"host": "shard-2.db", "port": 5432, "dbname": "app"},
            {"host": "shard-3.db", "port": 5432, "dbname": "app"},
        ]
        """
        self.num_shards = len(shard_configs)
        self.connections = {}
        for i, config in enumerate(shard_configs):
            self.connections[i] = psycopg2.connect(**config)
    
    def get_shard(self, shard_key: str) -> int:
        """Hash-based shard selection"""
        hash_value = int(hashlib.md5(str(shard_key).encode()).hexdigest(), 16)
        return hash_value % self.num_shards
    
    def get_connection(self, shard_key: str):
        """Get the database connection for a given shard key"""
        shard_id = self.get_shard(shard_key)
        return self.connections[shard_id]
    
    def query_single_shard(self, shard_key: str, sql: str, params: tuple):
        """Query a specific shard (fast - single shard)"""
        conn = self.get_connection(shard_key)
        cur = conn.cursor()
        cur.execute(sql, params)
        return cur.fetchall()
    
    def query_all_shards(self, sql: str, params: tuple = ()):
        """Scatter-gather query across ALL shards (slow but necessary)"""
        all_results = []
        for shard_id, conn in self.connections.items():
            cur = conn.cursor()
            cur.execute(sql, params)
            results = cur.fetchall()
            all_results.extend(results)
        return all_results


# ─── Usage ────────────────────────────────────────────────
router = ShardRouter([
    {"host": "shard-0.db.internal", "port": 5432, "dbname": "users"},
    {"host": "shard-1.db.internal", "port": 5432, "dbname": "users"},
    {"host": "shard-2.db.internal", "port": 5432, "dbname": "users"},
    {"host": "shard-3.db.internal", "port": 5432, "dbname": "users"},
])

# Fast: query with shard key
user = router.query_single_shard(
    shard_key="user_12345",
    sql="SELECT * FROM users WHERE user_id = %s",
    params=("user_12345",)
)

# Slow: query without shard key (scatter-gather)
recent_users = router.query_all_shards(
    sql="SELECT * FROM users WHERE created_at > %s ORDER BY created_at DESC LIMIT 10",
    params=("2024-01-01",)
)
# Note: Need to sort & limit the combined results in application!
```

### Java: Shard-Aware Repository

```java
// ─── Shard Router ────────────────────────────────────────
@Component
public class ShardRouter {
    
    private final List<DataSource> shards;
    private final int numShards;
    
    public ShardRouter(List<DataSource> shardDataSources) {
        this.shards = shardDataSources;
        this.numShards = shardDataSources.size();
    }
    
    public int getShardIndex(String shardKey) {
        // Consistent hash-based routing
        int hash = Math.abs(shardKey.hashCode());
        return hash % numShards;
    }
    
    public DataSource getDataSource(String shardKey) {
        return shards.get(getShardIndex(shardKey));
    }
    
    public List<DataSource> getAllShards() {
        return Collections.unmodifiableList(shards);
    }
}

// ─── Shard-Aware Repository ──────────────────────────────
@Repository
public class ShardedUserRepository {
    
    private final ShardRouter router;
    private final JdbcTemplate jdbcTemplate;
    
    // Single-shard query (fast!)
    public Optional<User> findById(String userId) {
        DataSource ds = router.getDataSource(userId);
        JdbcTemplate jt = new JdbcTemplate(ds);
        
        List<User> users = jt.query(
            "SELECT * FROM users WHERE user_id = ?",
            new UserRowMapper(),
            userId
        );
        return users.stream().findFirst();
    }
    
    // Cross-shard query (scatter-gather)
    public List<User> findByCountry(String country) {
        List<User> allResults = new ArrayList<>();
        
        // Query every shard in parallel
        List<CompletableFuture<List<User>>> futures = router.getAllShards()
            .stream()
            .map(ds -> CompletableFuture.supplyAsync(() -> {
                JdbcTemplate jt = new JdbcTemplate(ds);
                return jt.query(
                    "SELECT * FROM users WHERE country = ?",
                    new UserRowMapper(),
                    country
                );
            }))
            .toList();
        
        // Collect results from all shards
        futures.forEach(f -> allResults.addAll(f.join()));
        return allResults;
    }
    
    // Insert: route to correct shard
    public void save(User user) {
        DataSource ds = router.getDataSource(user.getId());
        JdbcTemplate jt = new JdbcTemplate(ds);
        
        jt.update(
            "INSERT INTO users (user_id, name, email, country) VALUES (?, ?, ?, ?)",
            user.getId(), user.getName(), user.getEmail(), user.getCountry()
        );
    }
}
```

---

## Rebalancing — Adding/Removing Shards

```
THE PROBLEM: You have 4 shards, need to add a 5th.
─────────────────────────────────────────────────────

With simple hash (key % N):
  Before: hash(key) % 4 = shard assignment
  After:  hash(key) % 5 = DIFFERENT shard assignment!
  
  ~80% of ALL data moves to different shards! 💀
  
With consistent hashing:
  Only ~20% of data moves (1/N fraction) ✅


REBALANCING STRATEGIES:
─────────────────────────────────────────────────────

Strategy 1: Fixed number of partitions (Cassandra, Riak)
┌─────────────────────────────────────────────────────┐
│  Create 256 virtual partitions from the start        │
│  Assign partitions to physical nodes:                │
│                                                       │
│  Node A: partitions 0-63                             │
│  Node B: partitions 64-127                           │
│  Node C: partitions 128-191                          │
│  Node D: partitions 192-255                          │
│                                                       │
│  Add Node E: Move some partitions from each node     │
│  Node A: 0-51                                        │
│  Node B: 64-115                                      │
│  Node C: 128-179                                     │
│  Node D: 192-243                                     │
│  Node E: 52-63, 116-127, 180-191, 244-255           │
│                                                       │
│  Data within a partition NEVER moves!                 │
│  Only partition → node assignment changes.            │
└─────────────────────────────────────────────────────┘

Strategy 2: Dynamic splitting (HBase, MongoDB)
┌─────────────────────────────────────────────────────┐
│  Start with 1 partition per node                     │
│  When a partition gets too large, SPLIT it:          │
│                                                       │
│  Before: Shard A [all data]                          │
│  Split:  Shard A [A-M], Shard B [N-Z]               │
│  Split:  Shard A [A-F], Shard B [G-M],              │
│          Shard C [N-S], Shard D [T-Z]               │
└─────────────────────────────────────────────────────┘
```

---

## Hot Spots — The Nemesis of Sharding

```
┌─────────────────────────────────────────────────────────────┐
│               HOT SPOT PROBLEM                               │
│                                                               │
│  Scenario: Social media app, shard key = user_id             │
│                                                               │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                   │
│  │Shard1│  │Shard2│  │Shard3│  │Shard4│                   │
│  │      │  │      │  │      │  │      │                   │
│  │ 10%  │  │ 10%  │  │ 70%  │  │ 10%  │  ← load          │
│  │ load │  │ load │  │ load │  │ load │                   │
│  │      │  │      │  │  🔥  │  │      │                   │
│  └──────┘  └──────┘  └──────┘  └──────┘                   │
│                                                               │
│  Why? Celebrity user on Shard 3 has 100M followers.          │
│  Every follower read hits Shard 3!                           │
│                                                               │
│  SOLUTIONS:                                                   │
│                                                               │
│  1. Add random suffix to hot keys:                           │
│     "celebrity_123" → "celebrity_123_01" to "celebrity_123_99"│
│     Spread across 99 shards! (but reads must aggregate)      │
│                                                               │
│  2. Cache hot data:                                          │
│     Put celebrity's followers in Redis                        │
│     Don't hit the shard at all                               │
│                                                               │
│  3. Replicate hot partitions:                                │
│     Read replicas for the hot shard specifically             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Infrastructure — Sharding Tools

```
┌─────────────────────────────────────────────────────────────┐
│              SHARDING INFRASTRUCTURE                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Application-Level Sharding:                                 │
│    • Your code decides routing                               │
│    • Full control, maximum complexity                        │
│                                                               │
│  Proxy-Based Sharding:                                       │
│    • Vitess (YouTube's MySQL sharding proxy)                 │
│    • ProxySQL                                                │
│    • Citus (PostgreSQL extension)                            │
│    App → Proxy → Correct shard                               │
│                                                               │
│  Native Sharding (built into the database):                  │
│    • MongoDB (automatic sharding)                            │
│    • Cassandra (consistent hashing built-in)                 │
│    • CockroachDB (automatic range-based)                     │
│    • Google Spanner (automatic partitioning)                 │
│    • DynamoDB (hash-based partitioning)                      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Instagram — Sharding PostgreSQL

Instagram shards their PostgreSQL by user_id:
- **Shard key**: user_id (all of a user's photos, likes, comments are on the same shard)
- **Strategy**: Pre-computed logical shards mapped to physical servers
- **Scale**: Thousands of logical shards across hundreds of physical servers
- Why user_id? Most queries are "show me this user's feed" or "show me this user's photos"

### Discord — Messages Sharded by Channel

Discord shards messages by `channel_id`:
- Each channel's messages are on one shard
- Reading a channel = single shard query (fast!)
- Cross-channel search = scatter-gather (slower, acceptable)
- They use Cassandra with `channel_id` as the partition key

### Notion — Sharded by Workspace

Notion shards by `workspace_id`:
- All pages in a workspace live on the same shard
- Enables fast within-workspace operations
- Cross-workspace features are handled separately

### Google Spanner — Automatic Geo-Sharding

Google Spanner automatically partitions data across global regions:
- You specify "this data should be near users in Europe"
- Spanner handles the rest (replication, consensus, rebalancing)
- Provides global consistency with TrueTime clocks

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Sharding too early | Massive complexity for no benefit | Wait until single DB is truly maxed out (~5TB, ~50K QPS) |
| Wrong shard key | Hot spots, or all queries are scatter-gather | Choose key that matches your access patterns |
| Cross-shard transactions | Extremely hard to implement correctly | Design data model to avoid cross-shard transactions |
| Cross-shard JOINs | Very slow, defeat the purpose of sharding | Denormalize; accept eventual consistency |
| Not planning for rebalancing | Can't add shards without downtime | Use consistent hashing or virtual partitions from day 1 |
| Shard key that changes | User moves to new country → must migrate data | Choose immutable shard keys |
| Auto-increment IDs across shards | Collisions! Shard 1 and 2 both have id=1 | Use UUIDs or shard-prefixed IDs (shard1_00001) |
| Uneven shard sizes | One shard is 10x bigger → becomes bottleneck | Monitor shard sizes; rebalance proactively |

---

## When to Use / When NOT to Use

### Use Data Partitioning When:
- Single database can't hold all data (>5-10 TB)
- Read/write throughput exceeds single machine limits
- You need geographic data locality (compliance, latency)
- Workload is naturally partitionable (multi-tenant, per-user)
- Availability requirements exceed single-machine guarantees

### Do NOT Partition When:
- Your data fits comfortably on one machine (<1 TB)
- You can scale vertically (bigger server) for less effort
- Most queries need data from multiple partitions
- You don't have the operational expertise to manage it
- Read replicas + caching can solve your performance issues

> **The #1 rule**: Sharding is a last resort. Exhaust all other scaling options first (read replicas, caching, query optimization, vertical scaling). Sharding adds permanent, irreversible complexity.

---

## Key Takeaways

1. **Sharding is the nuclear option** — Use it only when absolutely necessary. Once you shard, every feature you build must account for data being split across machines.

2. **The shard key is destiny** — Get it wrong and you'll either have hot spots (uneven load) or need scatter-gather for every query. Choose based on your most common access pattern.

3. **Consistent hashing solves rebalancing** — Simple `hash % N` breaks when N changes. Consistent hashing (or virtual partitions) ensures adding nodes only moves ~1/N of data.

4. **Hot spots will find you** — Even with perfect hashing, some keys are accessed 1000x more than others. Have a strategy: caching, key splitting, or dedicated replicas.

5. **Cross-shard operations are expensive** — Design your data model so that 95%+ of operations are single-shard. Accept that the remaining 5% will be slower.

6. **Modern databases handle this for you** — CockroachDB, Spanner, and DynamoDB manage partitioning automatically. Prefer managed solutions over DIY sharding.

7. **Monitor shard balance continuously** — Shards drift over time. What was balanced at launch becomes unbalanced as usage patterns change. Automate monitoring and rebalancing.

---

## What's Next?

Next, we'll explore **Multi-Tenancy Architecture** — how to design systems that serve multiple customers (tenants) from the same infrastructure while keeping their data isolated, secure, and performant.

See: [08-multi-tenancy.md](./08-multi-tenancy.md)
