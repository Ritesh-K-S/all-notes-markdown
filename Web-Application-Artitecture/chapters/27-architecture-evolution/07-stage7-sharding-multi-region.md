# Stage 7: Sharding + Multi-Region — 10M to 100M Users

> **What you'll learn**: How to split your database across multiple machines (sharding) and deploy your entire stack in multiple geographic regions, achieving massive scale and low latency for users worldwide.

---

## Real-Life Analogy

Your food court empire has grown into a **global chain**. You now have millions of customers across India, the US, Europe, and Southeast Asia.

**Problem 1 (Sharding)**: Your central recipe/inventory system has grown so large that one filing cabinet can't hold all the records. Solution: You split records **alphabetically** — customers A-H go to Cabinet 1, I-P to Cabinet 2, Q-Z to Cabinet 3. Each cabinet is on a different floor with its own manager.

**Problem 2 (Multi-Region)**: Customers in New York can't wait for orders to be processed in Mumbai (300ms delay). Solution: You open **full kitchens in each city** — Mumbai, New York, London, Singapore. Each city has its own kitchen, inventory, and staff. They sync data periodically.

---

## Core Concept Explained Step-by-Step

### What is Sharding?

Sharding splits one large database into multiple smaller databases (shards), each holding a subset of the data.

```
BEFORE: One massive database (hitting limits)
┌─────────────────────────────────────────┐
│          SINGLE DATABASE                 │
│                                         │
│  100 million users                       │
│  500 million orders                      │
│  2 TB data                              │
│  10,000 writes/second (MAXED OUT!)       │
│                                         │
│  RAM can't hold working set              │
│  Indexes don't fit in memory             │
│  Backup takes 12 hours                   │
└─────────────────────────────────────────┘

AFTER: Sharded across 4 databases
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  Shard 0   │ │  Shard 1   │ │  Shard 2   │ │  Shard 3   │
│            │ │            │ │            │ │            │
│ Users A-F  │ │ Users G-L  │ │ Users M-R  │ │ Users S-Z  │
│ 25M users  │ │ 25M users  │ │ 25M users  │ │ 25M users  │
│ 500 GB     │ │ 500 GB     │ │ 500 GB     │ │ 500 GB     │
│ 2500 w/s   │ │ 2500 w/s   │ │ 2500 w/s   │ │ 2500 w/s   │
└────────────┘ └────────────┘ └────────────┘ └────────────┘
                                    │
     Total: 10,000 writes/second distributed across 4 machines!
     Each shard easily handles its subset.
```

### Sharding Strategies

```
┌──────────────────────────────────────────────────────────────────────┐
│ STRATEGY 1: Hash-Based Sharding                                      │
│                                                                      │
│ shard_id = hash(user_id) % number_of_shards                         │
│                                                                      │
│ user_id=42:  hash(42) % 4 = 2  → Shard 2                           │
│ user_id=99:  hash(99) % 4 = 3  → Shard 3                           │
│ user_id=7:   hash(7)  % 4 = 3  → Shard 3                           │
│                                                                      │
│ ✓ Even distribution    ✗ Hard to add new shards (resharding)        │
├──────────────────────────────────────────────────────────────────────┤
│ STRATEGY 2: Range-Based Sharding                                     │
│                                                                      │
│ user_id 1-25M        → Shard 0                                      │
│ user_id 25M-50M      → Shard 1                                      │
│ user_id 50M-75M      → Shard 2                                      │
│ user_id 75M-100M     → Shard 3                                      │
│                                                                      │
│ ✓ Easy range queries    ✗ Hot spots (new users all go to last shard)│
├──────────────────────────────────────────────────────────────────────┤
│ STRATEGY 3: Geographic Sharding                                      │
│                                                                      │
│ India users    → Shard IN (Mumbai)                                   │
│ US users       → Shard US (Virginia)                                 │
│ Europe users   → Shard EU (Frankfurt)                                │
│ Asia users     → Shard AP (Singapore)                                │
│                                                                      │
│ ✓ Low latency for users  ✓ Data residency compliance  ✗ Uneven load│
└──────────────────────────────────────────────────────────────────────┘
```

### Multi-Region Architecture

```
                        GLOBAL DNS (Route53 / GeoDNS)
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
    User in India           User in US              User in Europe
            │                       │                       │
            ▼                       ▼                       ▼
┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
│  REGION: Mumbai   │  │  REGION: Virginia │  │ REGION: Frankfurt │
│                   │  │                   │  │                   │
│ ┌─────────────┐   │  │ ┌─────────────┐   │  │ ┌─────────────┐   │
│ │Load Balancer│   │  │ │Load Balancer│   │  │ │Load Balancer│   │
│ └──────┬──────┘   │  │ └──────┬──────┘   │  │ └──────┬──────┘   │
│        │          │  │        │          │  │        │          │
│ ┌──────▼──────┐   │  │ ┌──────▼──────┐   │  │ ┌──────▼──────┐   │
│ │ App Servers │   │  │ │ App Servers │   │  │ │ App Servers │   │
│ │ (10 inst.)  │   │  │ │ (20 inst.)  │   │  │ │ (8 inst.)   │   │
│ └──────┬──────┘   │  │ └──────┬──────┘   │  │ └──────┬──────┘   │
│        │          │  │        │          │  │        │          │
│ ┌──────▼──────┐   │  │ ┌──────▼──────┐   │  │ ┌──────▼──────┐   │
│ │   Redis     │   │  │ │   Redis     │   │  │ │   Redis     │   │
│ │  (Cache)    │   │  │ │  (Cache)    │   │  │ │  (Cache)    │   │
│ └──────┬──────┘   │  │ └──────┬──────┘   │  │ └──────┬──────┘   │
│        │          │  │        │          │  │        │          │
│ ┌──────▼──────┐   │  │ ┌──────▼──────┐   │  │ ┌──────▼──────┐   │
│ │ Database    │   │  │ │ Database    │   │  │ │ Database    │   │
│ │ (Primary)   │   │  │ │ (Primary)   │   │  │ │ (Primary)   │   │
│ └─────────────┘   │  │ └─────────────┘   │  │ └─────────────┘   │
└───────────────────┘  └───────────────────┘  └───────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    Cross-Region Replication
                    (async, 50-200ms delay)
```

### Why Multi-Region?

```
┌──────────────────────────────────────────────────────────────────┐
│                   LATENCY MATTERS                                  │
│                                                                  │
│  Same region (Mumbai → Mumbai):      ~5ms    ██                  │
│  Same continent (Mumbai → Singapore): ~50ms   ██████████         │
│  Cross-ocean (Mumbai → US-East):     ~200ms  ████████████████████│
│  Round trip (Mumbai → US → Mumbai):  ~400ms  (too slow!)         │
│                                                                  │
│  For a typical API call:                                         │
│  Without multi-region: Indian user → US server → 400ms RTT       │
│  With multi-region: Indian user → Mumbai server → 10ms RTT       │
│                                                                  │
│  That's 40x FASTER! Users feel the difference immediately.       │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Consistent Hashing (Smart Sharding)

Regular hashing breaks when you add/remove shards. Consistent hashing minimizes data movement:

```
Regular Hash (BAD):
hash(key) % 4 shards → If you add a 5th shard, ALMOST ALL keys move!
                        (massive reshuffling, downtime)

Consistent Hashing (GOOD):
Imagine a circle (0 to 2^32):

              0
          ╱       ╲
        ╱    S0     ╲
      ╱               ╲
    ╱    ●key1          ╲
   │         ●key2       │
   │  S3              S1 │
   │         ●key3       │
    ╲    ●key4          ╱
      ╲               ╱
        ╲    S2     ╱
          ╲       ╱
              
Each key goes to the NEXT shard clockwise on the ring.

Adding a new shard (S4): Only keys between S3 and S4 move!
                          (~1/N of all keys, not all of them)

┌─────────────────────────────────────────────────────────────┐
│  4 shards → 5 shards:                                       │
│  Regular hash: ~80% of keys must move (BAD!)                │
│  Consistent hash: ~20% of keys must move (GOOD!)            │
└─────────────────────────────────────────────────────────────┘
```

### Cross-Region Data Replication Strategies

```
STRATEGY 1: Active-Passive (One primary region)
┌──────────────────┐           ┌──────────────────┐
│  PRIMARY REGION  │  async    │ SECONDARY REGION │
│  (US-East)       │──────────▶│  (EU-West)       │
│                  │ replicate │                  │
│  ALL writes here │           │  Read-only       │
│  Read + Write    │           │  Reads only      │
└──────────────────┘           └──────────────────┘
✓ Simple      ✗ Writes from EU go to US (200ms extra latency)

STRATEGY 2: Active-Active (Multiple primary regions)
┌──────────────────┐           ┌──────────────────┐
│  REGION: US      │◀─────────▶│  REGION: EU      │
│                  │  bi-dir   │                  │
│  Read + Write    │  replicate│  Read + Write    │
│  (US users)      │           │  (EU users)      │
└──────────────────┘           └──────────────────┘
✓ Low latency everywhere    ✗ Conflict resolution needed!

CONFLICT EXAMPLE:
User updates email in US: "alice@new.com"  (at T=0)
User updates email in EU: "alice@other.com" (at T=0)
Both arrive at each other with delay...

Resolution strategies:
- Last Writer Wins (LWW): timestamp-based
- Application-level merge: custom logic
- CRDTs: conflict-free data structures
```

### How GeoDNS Routes Users

```
User in India types "myapp.com":

┌─────────────┐     ┌──────────────────────────────────────────┐
│ User's DNS  │────▶│  GeoDNS (Route53 / Cloudflare)            │
│ Resolver    │     │                                          │
│             │     │  IP: 103.x.x.x (India)                   │
│             │     │  Rule: If source IP is in India           │
│             │◀────│        → Return Mumbai LB IP              │
│             │     │        If source IP is in US              │
│  Gets:      │     │        → Return Virginia LB IP            │
│  Mumbai IP  │     │        If source IP is in EU              │
└─────────────┘     │        → Return Frankfurt LB IP           │
       │            └──────────────────────────────────────────┘
       │
       ▼
┌─────────────┐
│  Mumbai LB  │  ← User connects to nearest region (low latency!)
└─────────────┘
```

---

## Code Examples

### Python: Sharding Router

```python
# shard_router.py — Routes queries to the correct shard
import hashlib
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Define shard connections
SHARD_CONFIG = {
    0: "postgresql://user:pass@shard0.db.internal:5432/myapp",
    1: "postgresql://user:pass@shard1.db.internal:5432/myapp",
    2: "postgresql://user:pass@shard2.db.internal:5432/myapp",
    3: "postgresql://user:pass@shard3.db.internal:5432/myapp",
}

NUM_SHARDS = len(SHARD_CONFIG)

# Create engine pool for each shard
engines = {
    shard_id: create_engine(url, pool_size=10)
    for shard_id, url in SHARD_CONFIG.items()
}

def get_shard_id(user_id: int) -> int:
    """Determine which shard holds this user's data"""
    # Consistent hashing: same user_id ALWAYS goes to same shard
    hash_value = int(hashlib.md5(str(user_id).encode()).hexdigest(), 16)
    return hash_value % NUM_SHARDS

def get_session(user_id: int):
    """Get database session for the correct shard"""
    shard_id = get_shard_id(user_id)
    Session = sessionmaker(bind=engines[shard_id])
    return Session()

# Usage in API:
@app.route("/api/users/<int:user_id>")
def get_user(user_id):
    session = get_session(user_id)  # Routes to correct shard!
    try:
        user = session.query(User).filter_by(id=user_id).first()
        return jsonify(user.to_dict())
    finally:
        session.close()

@app.route("/api/users/<int:user_id>/orders")
def get_user_orders(user_id):
    session = get_session(user_id)  # Same shard has user's orders
    try:
        orders = session.query(Order).filter_by(user_id=user_id).all()
        return jsonify([o.to_dict() for o in orders])
    finally:
        session.close()

# CROSS-SHARD QUERY (expensive! avoid when possible)
@app.route("/api/admin/users/count")
def count_all_users():
    """Query ALL shards and aggregate (fan-out)"""
    total = 0
    for shard_id, engine in engines.items():
        Session = sessionmaker(bind=engine)
        session = Session()
        count = session.query(User).count()
        total += count
        session.close()
    return jsonify({"total_users": total})
```

### Java: Multi-Region Configuration with Spring Cloud

```java
// ShardRoutingDataSource.java — Routes to correct shard
@Configuration
public class ShardingConfig {

    private static final int NUM_SHARDS = 4;

    @Bean
    public Map<Integer, DataSource> shardDataSources() {
        Map<Integer, DataSource> shards = new HashMap<>();
        for (int i = 0; i < NUM_SHARDS; i++) {
            HikariConfig config = new HikariConfig();
            config.setJdbcUrl(String.format(
                "jdbc:postgresql://shard%d.db.internal:5432/myapp", i));
            config.setMaximumPoolSize(20);
            shards.put(i, new HikariDataSource(config));
        }
        return shards;
    }

    public static int getShardId(long userId) {
        // Consistent hash: same user always routes to same shard
        return Math.abs(Long.hashCode(userId)) % NUM_SHARDS;
    }
}

// UserRepository.java — Shard-aware queries
@Repository
public class ShardedUserRepository {

    @Autowired
    private Map<Integer, DataSource> shardDataSources;

    public User findById(long userId) {
        int shardId = ShardingConfig.getShardId(userId);
        DataSource ds = shardDataSources.get(shardId);

        try (Connection conn = ds.getConnection()) {
            PreparedStatement ps = conn.prepareStatement(
                "SELECT id, name, email FROM users WHERE id = ?");
            ps.setLong(1, userId);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                return new User(rs.getLong("id"), rs.getString("name"),
                               rs.getString("email"));
            }
            return null;
        }
    }

    // Fan-out query across all shards (expensive!)
    public long countAllUsers() {
        return shardDataSources.values().parallelStream()
            .mapToLong(ds -> {
                try (Connection conn = ds.getConnection();
                     Statement stmt = conn.createStatement();
                     ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM users")) {
                    rs.next();
                    return rs.getLong(1);
                } catch (SQLException e) {
                    throw new RuntimeException(e);
                }
            })
            .sum();
    }
}
```

---

## Infrastructure Example

### Multi-Region Deployment on AWS (Terraform)

```hcl
# multi-region.tf — Deploy to 3 regions

# Module for each region
module "us_east" {
  source = "./modules/region"
  region = "us-east-1"
  
  app_instance_count = 20
  db_instance_class  = "db.r6g.2xlarge"
  db_multi_az        = true
  
  # This region handles US users
  shard_range_start = 0
  shard_range_end   = 33  # Shards 0-33
}

module "eu_west" {
  source = "./modules/region"
  region = "eu-west-1"
  
  app_instance_count = 10
  db_instance_class  = "db.r6g.xlarge"
  db_multi_az        = true
  
  shard_range_start = 34
  shard_range_end   = 66  # Shards 34-66
}

module "ap_south" {
  source = "./modules/region"
  region = "ap-south-1"  # Mumbai
  
  app_instance_count = 15
  db_instance_class  = "db.r6g.xlarge"
  db_multi_az        = true
  
  shard_range_start = 67
  shard_range_end   = 99  # Shards 67-99
}

# Global DNS routing
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.com"
  type    = "A"

  # Latency-based routing: send users to nearest region
  latency_routing_policy {
    region = "us-east-1"
  }
  set_identifier = "us-east"
  alias {
    name    = module.us_east.alb_dns_name
    zone_id = module.us_east.alb_zone_id
  }
}

resource "aws_route53_record" "app_eu" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.com"
  type    = "A"

  latency_routing_policy {
    region = "eu-west-1"
  }
  set_identifier = "eu-west"
  alias {
    name    = module.eu_west.alb_dns_name
    zone_id = module.eu_west.alb_zone_id
  }
}

# Cross-region database replication
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "myapp-global"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  database_name             = "myapp"
}
```

### CockroachDB: Auto-Sharding + Multi-Region

```sql
-- CockroachDB handles sharding AND multi-region automatically!

-- Create a multi-region database
ALTER DATABASE myapp SET PRIMARY REGION "us-east1";
ALTER DATABASE myapp ADD REGION "eu-west1";
ALTER DATABASE myapp ADD REGION "ap-south1";

-- Table with regional data residency
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    email STRING NOT NULL,
    region crdb_internal_region NOT NULL,
    created_at TIMESTAMP DEFAULT now()
) LOCALITY REGIONAL BY ROW;

-- Data automatically lives in the user's region!
-- Indian users' data stored in ap-south1
-- US users' data stored in us-east1
-- Queries from India to Indian users: ~5ms (local!)
-- Queries from India to US users: ~200ms (cross-region)
```

### Vitess: MySQL Sharding (Used by YouTube/Slack)

```yaml
# vitess-schema.yaml — Sharding configuration
keyspaces:
  - name: users
    sharded: true
    vindexes:
      - name: user_id_hash
        type: hash
    tables:
      - name: users
        column_vindexes:
          - column: id
            name: user_id_hash
      - name: orders
        column_vindexes:
          - column: user_id
            name: user_id_hash
    # Co-locate user and their orders on same shard!
    # Queries JOIN users + orders are LOCAL (fast!)
```

---

## Real-World Example

### Instagram: Sharded PostgreSQL

Instagram shards their PostgreSQL database:
- **Shard key**: `user_id`
- **Thousands of shards** across many physical machines
- Photos, likes, comments — all sharded by the user who owns them
- Used a tool called **pgShard** (custom-built, later open-sourced concepts)

### Uber: Multi-Region Active-Active

Uber operates in **10,000+ cities** across the world:
- Each region runs independently
- Ride data stays in the region (Indian rides in Mumbai DC)
- Global services (pricing algorithms, ML models) replicated everywhere
- If one region fails, they can failover riders to the nearest healthy region

### Discord: Sharding by Guild (Server)

Discord handles billions of messages:
- Shard key: `guild_id` (Discord server)
- Each shard handles ~5,000 guilds
- Large guilds (like the official Fortnite server) get their own shard
- Messages within a guild never need cross-shard queries

### YouTube: Vitess

YouTube/Google uses **Vitess** to shard MySQL:
- Started as one MySQL database
- Grew to thousands of shards
- Vitess handles routing, resharding, and failover transparently
- Now handles millions of queries per second across thousands of MySQL instances

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Wrong shard key** | All traffic goes to one shard (hot shard) | Choose a key with even distribution (user_id, not country) |
| **Cross-shard queries** | Extremely slow (fan-out to all shards) | Design data model so common queries are single-shard |
| **Not planning for resharding** | Can't add shards later without downtime | Use consistent hashing or virtual shards |
| **Active-Active without conflict resolution** | Data corruption from concurrent writes | Implement LWW, vector clocks, or CRDTs |
| **Ignoring replication lag cross-region** | Users see stale data in remote regions | Accept eventual consistency, or route user to their home region |
| **No fallback for region failure** | Single region outage = all users in that region affected | Cross-region failover with health checks |
| **Over-sharding too early** | Complexity without need | Start with 1 DB, add read replicas, shard only when needed |

---

## When to Use / When NOT to Use

### ✅ Shard When:
- Single database can't hold all data (> 1-2 TB active data)
- Write throughput exceeds single-node capacity (> 10K writes/sec)
- Read replicas aren't enough (write-heavy workload)
- You need to isolate tenants (multi-tenant SaaS)
- Compliance requires data in specific regions (GDPR)

### ✅ Go Multi-Region When:
- Users are geographically distributed (multiple continents)
- Latency matters (< 100ms response time requirement)
- You need disaster recovery (survive entire region outage)
- Compliance requires data residency (GDPR: EU data stays in EU)
- You have > 10M users across multiple countries

### ❌ Don't Shard/Multi-Region When:
- Your data fits on one machine (< 500 GB)
- You have < 1M users concentrated in one geography
- The complexity cost outweighs the benefit
- Read replicas solve your read scaling problem
- You don't have the team/expertise to manage distributed data

---

## Key Takeaways

1. **Sharding splits data across machines** — each shard holds a subset, collectively they handle unlimited data
2. **Choose the shard key wisely** — it determines query efficiency; wrong key = hot spots
3. **Consistent hashing minimizes data movement** when adding/removing shards
4. **Multi-region reduces latency** — serve users from the nearest datacenter
5. **Active-Active is hard** — conflict resolution is a major challenge; start Active-Passive
6. **Cross-shard queries are expensive** — design your data model to avoid them
7. **This is the realm of specialized databases** — CockroachDB, Vitess, DynamoDB, Spanner handle this complexity for you

---

## What's Next?

With sharding and multi-region, you can handle 100M users. But what about billions? What about handling millions of requests per second? What about surviving cascading failures that could take down entire continents?

That's the final stage: **Planet-Scale Architecture** — Stage 8.

Next: [08-stage8-planet-scale.md](./08-stage8-planet-scale.md)
