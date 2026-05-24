# Key-Value Stores (Redis, DynamoDB)

> **What you'll learn**: How key-value stores work as the simplest and fastest type of database, when to use Redis vs DynamoDB, their internal architectures, and how companies use them for caching, sessions, leaderboards, and more.

---

## Real-Life Analogy

Think of a **coat check** at a fancy restaurant:

1. You hand over your coat → you get a **ticket number** (key)
2. When you want your coat back → you give the ticket → instant retrieval (value)

The attendant doesn't need to search through all coats by color or size. They just match your ticket number to the hook. **That's it. One key, one value, instant access.**

Now imagine this coat check can handle **millions of tickets per second** and retrieve any coat in **under 1 millisecond**. That's a key-value store.

---

## Core Concept Explained Step-by-Step

### The Simplest Database Model

```
KEY                          VALUE
─────────────────────────────────────────────────
"user:1001"          →       {"name": "Alice", "email": "alice@mail.co"}
"session:abc123"     →       {"user_id": 1001, "expires": 1705350000}
"cart:user:1001"     →       ["item_1", "item_2", "item_3"]
"rate_limit:ip:1.2"  →       47
"config:feature:dark" →      "enabled"
```

**That's the entire data model**: A key maps to a value. Period.

- **Key**: A unique string identifier (like a variable name)
- **Value**: Anything — string, number, JSON, binary blob, list, set, hash

### Why Key-Value Stores are So Fast

```
RELATIONAL DATABASE:                    KEY-VALUE STORE:
Query → Parse SQL                       Query → Hash(key)
     → Optimize plan                         → Direct memory lookup
     → Scan index                            → Return value
     → Read from disk                   
     → Filter results                   Time: ~0.1ms
     → Return rows                      
Time: ~5-50ms                           

```

Key-value stores skip ALL the complexity:
- No query parsing
- No query optimization
- No table scans or index traversals
- No joins or filtering
- Data often lives **entirely in RAM**

### Redis vs DynamoDB — Two Approaches

```
┌─────────────────────────────────────────────────────────────────┐
│                    KEY-VALUE STORES                               │
├───────────────────────────────┬─────────────────────────────────┤
│         REDIS                 │         DYNAMODB                 │
├───────────────────────────────┼─────────────────────────────────┤
│ In-memory (RAM-first)         │ On-disk (SSD-backed)            │
│ Single-threaded event loop    │ Distributed, managed service    │
│ Rich data structures          │ Simple key-value + document     │
│ Sub-millisecond latency       │ Single-digit millisecond        │
│ You manage infrastructure     │ AWS manages everything          │
│ Best for: Cache, sessions,    │ Best for: Serverless apps,      │
│   leaderboards, pub/sub       │   high-scale, predictable perf  │
│ Capacity: Limited by RAM      │ Capacity: Virtually unlimited   │
│ Persistence: Optional         │ Persistence: Always durable     │
│ Cost: RAM is expensive        │ Cost: Pay per request           │
└───────────────────────────────┴─────────────────────────────────┘
```

---

## How It Works Internally

### Redis Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    REDIS SERVER                            │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────┐                 │
│  │        Event Loop (Single Thread)    │                │
│  │   ┌────────┐  ┌────────┐  ┌──────┐ │                │
│  │   │ Accept │  │ Read   │  │Write │ │                │
│  │   │ Conn   │─▶│Command │─▶│Reply │ │                │
│  │   └────────┘  └────────┘  └──────┘ │                │
│  └─────────────────────────────────────┘                 │
│                      │                                    │
│                      ▼                                    │
│  ┌─────────────────────────────────────┐                 │
│  │     In-Memory Data Structures        │                │
│  │  ┌─────────┐ ┌──────┐ ┌──────────┐ │                │
│  │  │ Hash    │ │ Skip │ │ Linked   │ │                │
│  │  │ Tables  │ │ Lists│ │ Lists    │ │                │
│  │  └─────────┘ └──────┘ └──────────┘ │                │
│  └─────────────────────────────────────┘                 │
│                      │                                    │
│                      ▼ (Optional persistence)             │
│  ┌──────────────────┐  ┌───────────────────┐            │
│  │ RDB Snapshots    │  │ AOF (Append-Only  │            │
│  │ (Point-in-time)  │  │  File) Log        │            │
│  └──────────────────┘  └───────────────────┘            │
└──────────────────────────────────────────────────────────┘
```

**Why single-threaded?**
- No locks needed → no contention → no deadlocks
- All operations are atomic by default
- Event loop handles 100K+ operations/second on one core
- Network I/O is the bottleneck, not CPU (Redis 7+ uses I/O threads)

### Redis Data Structures

Redis isn't just strings — it has **rich data structures**:

```
┌──────────────────────────────────────────────────────────────────┐
│ STRING:  "user:1:name" → "Alice"                                 │
│          Simple value, counters, flags                            │
├──────────────────────────────────────────────────────────────────┤
│ HASH:    "user:1" → {name: "Alice", age: 28, city: "Mumbai"}    │
│          Like a mini-document / row                              │
├──────────────────────────────────────────────────────────────────┤
│ LIST:    "recent:posts" → [post5, post4, post3, post2, post1]   │
│          Ordered, push/pop from both ends (queue/stack)          │
├──────────────────────────────────────────────────────────────────┤
│ SET:     "user:1:friends" → {Alice, Bob, Charlie}               │
│          Unique values, set operations (union, intersect)        │
├──────────────────────────────────────────────────────────────────┤
│ SORTED SET: "leaderboard" → {Alice:1500, Bob:1200, Charlie:900} │
│          Scored members, range queries, rankings                  │
├──────────────────────────────────────────────────────────────────┤
│ STREAM:  "events" → [{id:1, data:"order_created"}, ...]        │
│          Append-only log, consumer groups (like mini Kafka)      │
└──────────────────────────────────────────────────────────────────┘
```

### DynamoDB Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     DynamoDB (AWS Managed)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐     ┌──────────────────────────────────┐      │
│  │ Request      │     │  Storage Nodes (SSDs)            │      │
│  │ Router       │────▶│                                  │      │
│  │              │     │  ┌────────┐ ┌────────┐ ┌──────┐ │      │
│  └──────────────┘     │  │Partition│ │Partition│ │Part. │ │      │
│        │              │  │   A    │ │   B    │ │  C   │ │      │
│        │              │  │(3 copies)│(3 copies)│(3 cop)│ │      │
│        ▼              │  └────────┘ └────────┘ └──────┘ │      │
│  ┌──────────────┐     └──────────────────────────────────┘      │
│  │ Partition    │                                                │
│  │ Metadata     │     Data automatically partitioned by          │
│  │ (Which key → │     partition key (hash)                       │
│  │  which node) │                                                │
│  └──────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘

Table Design:
┌────────────────────────────────────────────────────────────┐
│ Partition Key (PK) │ Sort Key (SK)    │ Attributes         │
├────────────────────┼──────────────────┼────────────────────┤
│ USER#alice         │ PROFILE          │ {name, email, ...} │
│ USER#alice         │ ORDER#2024-01-15 │ {items, total}     │
│ USER#alice         │ ORDER#2024-01-20 │ {items, total}     │
│ USER#bob           │ PROFILE          │ {name, email, ...} │
│ USER#bob           │ ORDER#2024-01-18 │ {items, total}     │
└────────────────────┴──────────────────┴────────────────────┘
```

**DynamoDB key concepts:**
- **Partition Key**: Determines which physical partition stores the data
- **Sort Key**: Orders items within a partition (enables range queries)
- **RCU/WCU**: Read/Write Capacity Units (you pay for throughput)
- **On-Demand**: Auto-scales, pay per request (simpler but costlier)

### Redis Persistence Options

```
RDB (Snapshot):                        AOF (Append-Only File):
┌─────────────────────────┐           ┌─────────────────────────┐
│ Full dump every N mins  │           │ Logs every write command │
│                         │           │                         │
│ Pro: Small file size    │           │ Pro: Minimal data loss  │
│ Pro: Fast restart       │           │ Pro: Human-readable log │
│ Con: Data loss between  │           │ Con: Larger file size   │
│      snapshots          │           │ Con: Slower restart     │
└─────────────────────────┘           └─────────────────────────┘

BOTH (Recommended for production):
RDB for fast restart + AOF for minimal data loss
```

---

## Code Examples

### Python (Redis)

```python
import redis
import json
from datetime import timedelta

# Connect to Redis
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# ━━━━ STRING: Caching API response ━━━━
def get_user_profile(user_id):
    cache_key = f"user:{user_id}:profile"
    
    # Try cache first
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache HIT — instant!
    
    # Cache miss — fetch from database
    profile = fetch_from_database(user_id)  # Slow: ~50ms
    
    # Store in cache with 1-hour expiry
    r.setex(cache_key, timedelta(hours=1), json.dumps(profile))
    return profile

# ━━━━ HASH: User session ━━━━
r.hset("session:abc123", mapping={
    "user_id": "1001",
    "role": "admin",
    "login_time": "2024-01-15T10:00:00Z"
})
r.expire("session:abc123", 3600)  # Expire in 1 hour

session = r.hgetall("session:abc123")
print(f"User: {session['user_id']}, Role: {session['role']}")

# ━━━━ SORTED SET: Real-time leaderboard ━━━━
r.zadd("leaderboard:game1", {"Alice": 1500, "Bob": 1200, "Charlie": 900})
r.zincrby("leaderboard:game1", 50, "Bob")  # Bob scores 50 more

# Top 10 players with scores
top_10 = r.zrevrange("leaderboard:game1", 0, 9, withscores=True)
for rank, (player, score) in enumerate(top_10, 1):
    print(f"#{rank}: {player} — {int(score)} pts")

# Player's rank
alice_rank = r.zrevrank("leaderboard:game1", "Alice")
print(f"Alice is rank #{alice_rank + 1}")

# ━━━━ LIST: Rate limiting (sliding window) ━━━━
import time

def is_rate_limited(user_id, max_requests=100, window_seconds=60):
    key = f"rate:{user_id}"
    now = time.time()
    
    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_seconds)  # Remove old entries
    pipe.zadd(key, {str(now): now})                       # Add current request
    pipe.zcard(key)                                        # Count requests
    pipe.expire(key, window_seconds)                       # Auto-cleanup
    results = pipe.execute()
    
    request_count = results[2]
    return request_count > max_requests
```

### Java (Redis with Jedis)

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;
import java.util.*;

public class RedisExample {
    private static JedisPool pool = new JedisPool(
        new JedisPoolConfig(), "localhost", 6379);

    public static void main(String[] args) {
        try (Jedis jedis = pool.getResource()) {
            // Cache with expiry
            jedis.setex("api:weather:mumbai", 300, "{\"temp\": 32, \"humidity\": 75}");
            String weather = jedis.get("api:weather:mumbai");
            System.out.println("Weather: " + weather);

            // Hash — user profile
            Map<String, String> userProfile = new HashMap<>();
            userProfile.put("name", "Bob");
            userProfile.put("email", "bob@mail.co");
            userProfile.put("plan", "premium");
            jedis.hset("user:2001", userProfile);
            
            String plan = jedis.hget("user:2001", "plan");
            System.out.println("Plan: " + plan);

            // Sorted Set — leaderboard
            jedis.zadd("scores", 1500, "Alice");
            jedis.zadd("scores", 1200, "Bob");
            jedis.zadd("scores", 900, "Charlie");
            jedis.zincrby("scores", 100, "Charlie");  // Charlie gains 100

            // Get top 3 with scores
            List<String> top3 = jedis.zrevrange("scores", 0, 2)
                .stream().toList();
            System.out.println("Top 3: " + top3);

            // Distributed lock (simplified)
            String lockKey = "lock:order:123";
            String lockValue = UUID.randomUUID().toString();
            String result = jedis.set(lockKey, lockValue, 
                SetParams.setParams().nx().ex(10));  // NX=only if not exists, EX=10s
            
            if ("OK".equals(result)) {
                System.out.println("Lock acquired! Processing order...");
                // ... do work ...
                jedis.del(lockKey);  // Release lock
            }
        }
    }
}
```

### Python (DynamoDB with boto3)

```python
import boto3
from boto3.dynamodb.conditions import Key

# Connect to DynamoDB
dynamodb = boto3.resource('dynamodb', region_name='ap-south-1')
table = dynamodb.Table('UserOrders')

# Single-table design: store users and orders together
# Put user profile
table.put_item(Item={
    'PK': 'USER#alice',
    'SK': 'PROFILE',
    'name': 'Alice Johnson',
    'email': 'alice@example.com',
    'plan': 'premium'
})

# Put order
table.put_item(Item={
    'PK': 'USER#alice',
    'SK': 'ORDER#2024-01-15#ord_123',
    'items': [{'name': 'Laptop', 'price': 999}],
    'total': 999,
    'status': 'delivered'
})

# Query: Get all of Alice's data (profile + all orders)
response = table.query(
    KeyConditionExpression=Key('PK').eq('USER#alice')
)
for item in response['Items']:
    print(f"  {item['SK']}: {item}")

# Query: Get only Alice's orders from January 2024
response = table.query(
    KeyConditionExpression=Key('PK').eq('USER#alice') & 
                           Key('SK').begins_with('ORDER#2024-01')
)
```

---

## Infrastructure Examples

### Redis Cluster (Docker Compose)

```yaml
version: '3.8'
services:
  redis-node-1:
    image: redis:7
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --port 6379
    ports:
      - "6379:6379"
  redis-node-2:
    image: redis:7
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --port 6379
    ports:
      - "6380:6379"
  redis-node-3:
    image: redis:7
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --port 6379
    ports:
      - "6381:6379"
```

### DynamoDB Table (Terraform)

```hcl
resource "aws_dynamodb_table" "user_orders" {
  name           = "UserOrders"
  billing_mode   = "PAY_PER_REQUEST"  # On-demand scaling
  hash_key       = "PK"
  range_key      = "SK"

  attribute {
    name = "PK"
    type = "S"
  }
  attribute {
    name = "SK"
    type = "S"
  }
  attribute {
    name = "GSI1PK"
    type = "S"
  }

  # Global Secondary Index for different access pattern
  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "SK"
    projection_type = "ALL"
  }

  point_in_time_recovery { enabled = true }
  server_side_encryption { enabled = true }

  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }
}
```

---

## Real-World Example

### Twitter/X — Redis for Timeline Caching

```
┌─────────────────────────────────────────────────────────────┐
│              Twitter Timeline Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User tweets ──▶ Fan-out Service ──▶ Write to Redis Lists   │
│                                                              │
│  For each follower:                                         │
│    LPUSH timeline:{follower_id} {tweet_id}                  │
│    LTRIM timeline:{follower_id} 0 799  (keep last 800)      │
│                                                              │
│  User opens app:                                            │
│    LRANGE timeline:{user_id} 0 19  (get latest 20 tweets)  │
│    → Sub-millisecond response!                              │
│                                                              │
│  ┌────────────────┐    ┌─────────────┐    ┌──────────────┐ │
│  │ Timeline Cache │    │ User Cache  │    │ Tweet Cache  │ │
│  │ (Redis Lists)  │    │ (Redis Hash)│    │ (Redis Hash) │ │
│  │ 300M+ keys    │    │ Profiles    │    │ Content      │ │
│  └────────────────┘    └─────────────┘    └──────────────┘ │
│                                                              │
│  Redis Cluster: 1000+ nodes, 100TB+ RAM                     │
└─────────────────────────────────────────────────────────────┘
```

### Amazon — DynamoDB Powers Core Services

- **Amazon.com shopping cart**: DynamoDB handles millions of cart operations/second
- **Prime Video**: User watch history, preferences
- **Alexa**: Skill state, user context
- **All internal services**: DynamoDB is Amazon's default database choice

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using Redis as primary database | Data loss if server crashes (without persistence) | Enable AOF + use replicas |
| No TTL on cache keys | Memory fills up, OOM kills Redis | Always set expiration |
| Hot keys in DynamoDB | Throttling (one partition gets all traffic) | Add randomness to keys, use DAX |
| Large values (>1MB) | Blocks Redis event loop, network slow | Compress or split data |
| Not using connection pools | Opening/closing connections per request | Use pool (JedisPool, redis-py pool) |
| `KEYS *` in production | Scans ALL keys, blocks everything | Use `SCAN` for iteration |
| Storing everything in Redis | RAM is 10x more expensive than disk | Cache hot data, archive cold data |

---

## When to Use / When NOT to Use

### ✅ Use Key-Value Stores When:
- You need **sub-millisecond** read latency (caching)
- Access pattern is **get by exact key** (sessions, configs)
- Data has natural **TTL** (sessions expire, cache refreshes)
- Simple **counters or rate limiting** (no complex queries)
- **Leaderboards or rankings** (Redis sorted sets)
- **Message queues or pub/sub** (Redis Streams)
- **Serverless applications** needing auto-scaling storage (DynamoDB)

### ❌ Do NOT Use Key-Value Stores When:
- You need **complex queries** (filter, sort, JOIN, aggregate)
- **Relationships between data** are important
- You need **full-text search**
- Data exceeds available **RAM** (Redis) without clear cold/hot separation
- You need **ad-hoc queries** (analytics, reporting)
- **Strong consistency** across multiple keys is required (use SQL)

---

## Key Takeaways

1. **Key-value stores** are the simplest database model — one key maps to one value, with O(1) lookup time.
2. **Redis** lives in RAM, provides sub-millisecond latency, and offers rich data structures (hashes, lists, sorted sets, streams).
3. **DynamoDB** is a fully managed, infinitely scalable key-value store — pay per request, no infrastructure to manage.
4. **Use Redis for**: caching, sessions, leaderboards, rate limiting, pub/sub.
5. **Use DynamoDB for**: serverless apps, high-scale CRUD, when you need guaranteed single-digit ms latency at any scale.
6. **Always set TTL** on cache entries — unbounded data growth will eventually crash your Redis instance.
7. **Design DynamoDB tables around access patterns** — single-table design with PK/SK is key to avoiding expensive scans.

---

## What's Next?

Next, we'll explore **Column-Family Stores (Cassandra, HBase)** (Chapter 9.5), databases designed for massive write throughput and time-series data at planet scale.
