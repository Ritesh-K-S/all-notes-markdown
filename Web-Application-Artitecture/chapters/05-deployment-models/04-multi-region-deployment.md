# Multi-Region Deployment — Serving Users Across the Globe

> **What you'll learn**: How to deploy your application in multiple geographic regions simultaneously — so a user in Tokyo, Mumbai, London, and New York all get fast responses — including the brutal challenges of data replication, conflict resolution, and failover.

---

## Real-Life Analogy

Imagine you own a pizza chain. If you have **one kitchen in New York**, customers in London wait 6 hours for delivery (the "latency"). That's unacceptable.

Solution: Open **kitchens in every major city** — New York, London, Mumbai, Tokyo. Each kitchen:
- Has its own ingredients (data)
- Makes the same menu (same application code)
- Serves local customers instantly
- Syncs recipes and inventory updates between locations (data replication)

If the London kitchen burns down (server failure), London customers get routed to the nearest open kitchen (Paris/Amsterdam) — maybe slightly slower, but they still get their pizza. That's **multi-region deployment with failover**.

---

## Core Concept Explained Step-by-Step

### Why Multi-Region?

```
SINGLE REGION (US-East):

User in India ───── 200ms ────▶ US-East Server ───── 200ms ────▶ Response
                              (Round trip: ~400ms)

User in US ──────── 20ms ─────▶ US-East Server ──── 20ms ─────▶ Response
                              (Round trip: ~40ms)

The Indian user ALWAYS gets 10x worse experience! 😞


MULTI-REGION (US-East + Asia + Europe):

User in India ───── 20ms ─────▶ Mumbai Server ───── 20ms ─────▶ Response ✓
User in US ──────── 20ms ─────▶ US-East Server ──── 20ms ─────▶ Response ✓
User in UK ──────── 15ms ─────▶ London Server ───── 15ms ─────▶ Response ✓

EVERYONE gets fast responses! 🎉
```

### Latency by Distance (Speed of Light Limit)

```
┌─────────────────────────────────────────────────────────┐
│  PHYSICAL DISTANCE = MINIMUM LATENCY                    │
│  (Speed of light in fiber ≈ 200,000 km/s)              │
│                                                         │
│  Route                        │ Distance │ Min Latency  │
│  ─────────────────────────────┼──────────┼────────────  │
│  Same datacenter              │  <1 km   │   <1 ms      │
│  Same city                    │  50 km   │   ~1 ms      │
│  US East ↔ US West           │  4000 km │   ~40 ms     │
│  US East ↔ Europe            │  6000 km │   ~60 ms     │
│  US East ↔ India             │  13000 km│   ~130 ms    │
│  US East ↔ Australia         │  16000 km│   ~160 ms    │
│                                                         │
│  These are MINIMUM (speed of light). Real-world is      │
│  1.5-3x worse due to routing hops and processing.      │
└─────────────────────────────────────────────────────────┘
```

### The Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     MULTI-REGION ARCHITECTURE                           │
│                                                                         │
│  ┌───────────────┐                                                     │
│  │  Global DNS   │  (Route 53 / Cloudflare GeoDNS)                    │
│  │  GeoDNS       │  Routes users to NEAREST region                     │
│  └───────┬───────┘                                                     │
│          │                                                             │
│    ┌─────┼─────────────────────────────────────┐                       │
│    │     │              │                      │                       │
│    ▼     ▼              ▼                      ▼                       │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │  US-EAST    │  │  EU-WEST    │  │  AP-SOUTH   │  │  AP-EAST    │  │
│  │  (Virginia) │  │  (Ireland)  │  │  (Mumbai)   │  │  (Tokyo)    │  │
│  │             │  │             │  │             │  │             │  │
│  │  ┌───────┐ │  │  ┌───────┐ │  │  ┌───────┐ │  │  ┌───────┐ │  │
│  │  │  LB   │ │  │  │  LB   │ │  │  │  LB   │ │  │  │  LB   │ │  │
│  │  └───┬───┘ │  │  └───┬───┘ │  │  └───┬───┘ │  │  └───┬───┘ │  │
│  │      │     │  │      │     │  │      │     │  │      │     │  │
│  │  ┌───┴───┐ │  │  ┌───┴───┐ │  │  ┌───┴───┐ │  │  ┌───┴───┐ │  │
│  │  │ App×5 │ │  │  │ App×3 │ │  │  │ App×4 │ │  │  │ App×3 │ │  │
│  │  └───┬───┘ │  │  └───┬───┘ │  │  └───┬───┘ │  │  └───┬───┘ │  │
│  │      │     │  │      │     │  │      │     │  │      │     │  │
│  │  ┌───┴───┐ │  │  ┌───┴───┐ │  │  ┌───┴───┐ │  │  ┌───┴───┐ │  │
│  │  │  DB   │ │  │  │  DB   │ │  │  │  DB   │ │  │  │  DB   │ │  │
│  │  │PRIMARY│ │  │  │REPLICA│ │  │  │REPLICA│ │  │  │REPLICA│ │  │
│  │  └───────┘ │  │  └───────┘ │  │  └───────┘ │  │  └───────┘ │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │
│                                                                         │
│        ◀══════════════ DATA REPLICATION ══════════════▶                 │
│                   (async, ~100-500ms lag)                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### GeoDNS — Routing Users to the Nearest Region

```
USER IN INDIA types "myapp.com" in browser:

┌──────────┐      ┌───────────────────────────────────────┐
│  User    │─────▶│  DNS Resolver                         │
│ (Mumbai) │      │                                       │
└──────────┘      │  Query: myapp.com                     │
                  │  User IP: 49.37.xx.xx (India)         │
                  └───────────────────┬───────────────────┘
                                      │
                                      ▼
                  ┌───────────────────────────────────────┐
                  │  Route 53 (GeoDNS)                    │
                  │                                       │
                  │  Rules:                               │
                  │  IP from Asia  → ap-south-1 endpoint  │
                  │  IP from EU    → eu-west-1 endpoint   │
                  │  IP from US    → us-east-1 endpoint   │
                  │  Default       → us-east-1 (primary)  │
                  │                                       │
                  │  User is from India → return:         │
                  │  myapp-mumbai.amazonaws.com            │
                  └───────────────────────────────────────┘

Result: User connects to Mumbai region (20ms) instead of US-East (200ms)!
```

### Data Replication Strategies

```
STRATEGY 1: Primary-Replica (Read from local, Write to primary)

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  US-EAST (PRIMARY):           EU-WEST (REPLICA):                    │
│  ┌───────────────┐            ┌───────────────┐                    │
│  │  PostgreSQL   │───ASYNC───▶│  PostgreSQL   │                    │
│  │  (read+write) │  replicate │  (read-only)  │                    │
│  └───────────────┘            └───────────────┘                    │
│                                                                     │
│  EU user reads: → local replica (fast! 15ms)                       │
│  EU user writes: → forwarded to US-East primary (slow! 130ms)      │
│                                                                     │
│  PRO: Simple, consistent writes                                    │
│  CON: Writes are slow from non-primary regions                     │
└─────────────────────────────────────────────────────────────────────┘


STRATEGY 2: Multi-Primary (Write locally, sync everywhere)

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  US-EAST (PRIMARY):           EU-WEST (PRIMARY):                    │
│  ┌───────────────┐            ┌───────────────┐                    │
│  │  CockroachDB  │◀──SYNC───▶│  CockroachDB  │                    │
│  │  (read+write) │  replicate │  (read+write) │                    │
│  └───────────────┘            └───────────────┘                    │
│                                                                     │
│  EU user reads: → local (fast!)                                    │
│  EU user writes: → local (fast!) but must resolve conflicts        │
│                                                                     │
│  PRO: Fast reads AND writes everywhere                             │
│  CON: Conflict resolution is HARD. Data might be stale briefly.    │
│  TOOLS: CockroachDB, Google Spanner, DynamoDB Global Tables        │
└─────────────────────────────────────────────────────────────────────┘


STRATEGY 3: Eventual Consistency (Write anywhere, sync later)

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  US-EAST:            EU-WEST:            AP-SOUTH:                  │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐               │
│  │DynamoDB │◀──────▶│DynamoDB │◀──────▶│DynamoDB │               │
│  │ (write) │  async │ (write) │  async │ (write) │               │
│  └─────────┘        └─────────┘        └─────────┘               │
│                                                                     │
│  All regions can write. Conflicts resolved by "last writer wins"   │
│  or application-level merge.                                        │
│                                                                     │
│  PRO: Maximum availability, lowest latency everywhere              │
│  CON: Brief inconsistency (user might see stale data for seconds)  │
│  USE CASE: Shopping cart, user preferences, social media likes      │
└─────────────────────────────────────────────────────────────────────┘
```

### Failover — When a Region Goes Down

```
NORMAL OPERATION:
┌──────────┐                    ┌──────────┐
│  Users   │────GeoDNS─────▶  │ US-EAST  │  (healthy ✓)
│  (US)    │                    └──────────┘
└──────────┘

┌──────────┐                    ┌──────────┐
│  Users   │────GeoDNS─────▶  │ EU-WEST  │  (healthy ✓)
│  (EU)    │                    └──────────┘
└──────────┘


FAILURE: US-EAST goes down!
┌──────────┐                    ┌──────────┐
│  Users   │────GeoDNS──╳──▶  │ US-EAST  │  (DEAD ✗)
│  (US)    │                    └──────────┘
└──────────┘

FAILOVER: GeoDNS detects failure, reroutes US users to EU-WEST:
┌──────────┐                    ┌──────────┐
│  Users   │────GeoDNS─────▶  │ EU-WEST  │  (handles both!)
│  (US)    │                    └──────────┘
└──────────┘
                                Latency goes from 20ms → 80ms
                                (degraded but STILL WORKING!)

RECOVERY: US-EAST comes back, traffic gradually returns.
```

### Active-Active vs Active-Passive

```
ACTIVE-ACTIVE (all regions serve traffic simultaneously):
┌─────────┐  ┌─────────┐  ┌─────────┐
│ US-EAST │  │ EU-WEST │  │AP-SOUTH │
│(serving)│  │(serving)│  │(serving)│  ← ALL active, all receiving traffic
└─────────┘  └─────────┘  └─────────┘
Used by: Netflix, Google, Amazon
PRO: Best performance, uses all capacity
CON: Data sync is complex, conflicts possible


ACTIVE-PASSIVE (one primary, others are standby):
┌─────────┐  ┌─────────┐  ┌─────────┐
│ US-EAST │  │ EU-WEST │  │AP-SOUTH │
│(ACTIVE) │  │(standby)│  │(standby)│  ← Only one serves traffic
└─────────┘  └─────────┘  └─────────┘
Used by: Smaller companies, simpler setup
PRO: No conflict resolution needed (one write location)
CON: Other regions are wasted capacity (pay for idle servers)
```

---

## Code Examples

### Python (Region-Aware Application)

```python
# app.py — Application that knows which region it's in
from flask import Flask, jsonify, request
import os
import psycopg2
from functools import lru_cache

app = Flask(__name__)

REGION = os.environ.get("AWS_REGION", "us-east-1")  # Which region am I in?

# Database configuration per region
DB_CONFIG = {
    "us-east-1": {"host": "db-primary.us-east-1.rds.amazonaws.com", "read_write": True},
    "eu-west-1": {"host": "db-replica.eu-west-1.rds.amazonaws.com", "read_write": False},
    "ap-south-1": {"host": "db-replica.ap-south-1.rds.amazonaws.com", "read_write": False},
}

PRIMARY_REGION = "us-east-1"
PRIMARY_DB = DB_CONFIG[PRIMARY_REGION]["host"]

def get_local_db():
    """Connect to the local region's database (fast reads)."""
    config = DB_CONFIG[REGION]
    return psycopg2.connect(host=config["host"], dbname="myapp",
                           user=os.environ["DB_USER"], password=os.environ["DB_PASS"])

def get_primary_db():
    """Connect to primary database (for writes)."""
    return psycopg2.connect(host=PRIMARY_DB, dbname="myapp",
                           user=os.environ["DB_USER"], password=os.environ["DB_PASS"])

@app.route("/api/products")
def get_products():
    """READS go to the LOCAL replica (fast!)."""
    conn = get_local_db()  # Read from local region's DB
    cur = conn.cursor()
    cur.execute("SELECT id, name, price FROM products")
    products = [{"id": r[0], "name": r[1], "price": float(r[2])} for r in cur.fetchall()]
    conn.close()
    return jsonify({"products": products, "served_from": REGION})

@app.route("/api/orders", methods=["POST"])
def create_order():
    """WRITES go to the PRIMARY database (may be in another region)."""
    data = request.json
    conn = get_primary_db()  # Write to primary (might be cross-region!)
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO orders (user_id, product_id, quantity) VALUES (%s, %s, %s) RETURNING id",
        (data["user_id"], data["product_id"], data["quantity"])
    )
    order_id = cur.fetchone()[0]
    conn.commit()
    conn.close()
    return jsonify({"order_id": order_id, "written_to": PRIMARY_REGION}), 201

@app.route("/health")
def health():
    return jsonify({"status": "healthy", "region": REGION})
```

### Java (Spring Boot with Region-Aware Data Source Routing)

```java
// RegionAwareDataSource.java — Route reads locally, writes to primary
@Configuration
public class DataSourceConfig {
    
    @Value("${app.region}")
    private String currentRegion;
    
    @Bean
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("READ", createDataSource(getLocalDbHost()));
        targetDataSources.put("WRITE", createDataSource(getPrimaryDbHost()));
        
        AbstractRoutingDataSource routingDataSource = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                // If in a @Transactional(readOnly=true) context → route to local replica
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly() 
                    ? "READ" : "WRITE";
            }
        };
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(targetDataSources.get("READ"));
        return routingDataSource;
    }
    
    private String getLocalDbHost() {
        // Read from local replica (fast!)
        return switch (currentRegion) {
            case "us-east-1" -> "db-primary.us-east-1.rds.amazonaws.com";
            case "eu-west-1" -> "db-replica.eu-west-1.rds.amazonaws.com";
            case "ap-south-1" -> "db-replica.ap-south-1.rds.amazonaws.com";
            default -> "db-primary.us-east-1.rds.amazonaws.com";
        };
    }
    
    private String getPrimaryDbHost() {
        // All writes go to primary (us-east-1)
        return "db-primary.us-east-1.rds.amazonaws.com";
    }
}

// Usage in service:
@Service
public class ProductService {
    @Transactional(readOnly = true)  // ← Routed to LOCAL replica
    public List<Product> getProducts() {
        return productRepository.findAll();
    }
    
    @Transactional  // ← Routed to PRIMARY (cross-region if needed)
    public Order createOrder(OrderRequest request) {
        return orderRepository.save(new Order(request));
    }
}
```

---

## Infrastructure Example

### AWS Multi-Region Setup

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          AWS MULTI-REGION                                   │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                    Route 53 (Global DNS)                            │    │
│  │  Routing Policy: Latency-based                                     │    │
│  │  Health Checks: Every 30s per region                               │    │
│  └──────┬─────────────────────┬───────────────────────┬──────────────┘    │
│         │                     │                       │                     │
│         ▼                     ▼                       ▼                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐              │
│  │  US-EAST-1   │     │  EU-WEST-1   │     │ AP-SOUTH-1   │              │
│  │  (Virginia)  │     │  (Ireland)   │     │  (Mumbai)    │              │
│  │              │     │              │     │              │              │
│  │  ALB         │     │  ALB         │     │  ALB         │              │
│  │  ├─ ECS ×5  │     │  ├─ ECS ×3  │     │  ├─ ECS ×4  │              │
│  │  ├─ Redis   │     │  ├─ Redis   │     │  ├─ Redis   │              │
│  │  └─ RDS     │     │  └─ RDS     │     │  └─ RDS     │              │
│  │    (PRIMARY)│     │   (REPLICA) │     │   (REPLICA) │              │
│  └──────────────┘     └──────────────┘     └──────────────┘              │
│         │                     ▲                       ▲                     │
│         │                     │                       │                     │
│         └─────── Async Replication ──────────────────┘                    │
│                   (lag: 50-200ms)                                           │
│                                                                             │
│  Global Services:                                                          │
│  ├─ CloudFront CDN (static assets cached at 200+ edge locations)          │
│  ├─ DynamoDB Global Tables (user sessions — replicated everywhere)        │
│  └─ S3 Cross-Region Replication (uploaded files available globally)        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Terraform Multi-Region

```hcl
# main.tf — Deploy same app to multiple AWS regions
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_south"
  region = "ap-south-1"
}

# Primary database in US-East
resource "aws_db_instance" "primary" {
  provider      = aws.us_east
  engine        = "postgres"
  instance_class = "db.r5.xlarge"
  multi_az      = true  # High availability within region
  
  backup_retention_period = 7
}

# Read replica in EU-West
resource "aws_db_instance" "replica_eu" {
  provider             = aws.eu_west
  replicate_source_db  = aws_db_instance.primary.arn
  instance_class       = "db.r5.large"
}

# Read replica in Asia
resource "aws_db_instance" "replica_asia" {
  provider             = aws.ap_south
  replicate_source_db  = aws_db_instance.primary.arn
  instance_class       = "db.r5.large"
}

# Route 53 latency-based routing
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.myapp.com"
  type    = "A"
  
  set_identifier = "us-east"
  latency_routing_policy {
    region = "us-east-1"
  }
  
  alias {
    name    = aws_lb.us_east.dns_name
    zone_id = aws_lb.us_east.zone_id
  }
}

# (Repeat for eu-west-1 and ap-south-1)
```

---

## Real-World Example

### Netflix — Across 3 AWS Regions

```
Netflix Architecture (simplified):

┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  Regions: US-East, US-West, EU-West                          │
│                                                               │
│  Each region has:                                             │
│  ├── 1000+ EC2 instances (microservices)                     │
│  ├── Own Cassandra cluster (multi-region replication)         │
│  ├── Own ElastiCache (Redis) cluster                         │
│  ├── Own EVCache (memcached) layer                           │
│  └── Full copy of the streaming catalog                      │
│                                                               │
│  Data Replication:                                            │
│  ├── Cassandra: Eventual consistency (tunable per query)     │
│  ├── Video files: Pre-positioned on CDN (Open Connect)       │
│  └── User data: Replicated across all 3 regions              │
│                                                               │
│  Failover tested regularly:                                   │
│  "We ROUTINELY fail over an entire region to test            │
│   that our systems survive." — Netflix engineering blog       │
└───────────────────────────────────────────────────────────────┘
```

### Google — Global Spanner Database

```
Google Spanner: A globally distributed database that provides:
- Strong consistency across continents
- TrueTime (GPS + atomic clocks for global ordering)
- Automatic sharding and replication

┌──────────┐     ┌──────────┐     ┌──────────┐
│  US      │◀───▶│  Europe  │◀───▶│  Asia    │
│  Shard 1 │     │  Shard 2 │     │  Shard 3 │
└──────────┘     └──────────┘     └──────────┘

A write in the US is visible in Asia within ~7ms (TrueTime guarantee).
No other database can do this.

Used for: Google Ads, Google Play, Google Finance
```

### Shopify — Active-Active Multi-Region

```
Shopify (2022+) moved to active-active across 3 regions:

Problem: Black Friday traffic is 50x normal
- Single region couldn't handle the spike
- If primary region failed on Black Friday = catastrophe

Solution:
- 3 regions, each handling ~33% of merchants
- If one region fails, other two absorb its traffic
- Database: Vitess (sharded MySQL) with cross-region replication
- All regions can write (for their assigned merchants)
```

---

## Common Mistakes / Pitfalls

### 1. Ignoring Replication Lag
❌ **Mistake**: User creates an order in US-East, immediately reads it from EU-replica — NOT THERE YET (200ms lag).
✅ **Fix**: Read-your-own-writes guarantee — after a write, read from primary (or use session stickiness to same region).

```python
# After a write, redirect to primary for the next read
@app.route("/api/orders", methods=["POST"])
def create_order():
    conn = get_primary_db()
    # ... create order ...
    # Return with header telling client to read from primary
    resp = jsonify({"order_id": order_id})
    resp.headers["X-Read-From"] = "primary"  # Client/proxy uses this
    return resp
```

### 2. No Global Health Checks
❌ **Mistake**: Region goes down but DNS still routes users there → users get timeouts.
✅ **Fix**: Configure DNS health checks that probe each region every 30s and remove unhealthy regions.

### 3. Hardcoded Region Assumptions
❌ **Mistake**: Code assumes "I'm always in US-East" — breaks when deployed to another region.
✅ **Fix**: Read region from environment variable, configure everything dynamically.

### 4. Synchronous Cross-Region Calls
❌ **Mistake**: Service in EU calls a service in US synchronously → 130ms added to EVERY request.
✅ **Fix**: Keep all synchronous calls within the same region. Use async replication for cross-region data.

### 5. Not Testing Failover
❌ **Mistake**: "We have multi-region!" but never tested what happens when one goes down.
✅ **Fix**: Regularly perform failover drills (Netflix's "Chaos Engineering" approach — see Chapter 12.9).

---

## When to Use / When NOT to Use

### ✅ Use Multi-Region When:

| Criteria | Why |
|----------|-----|
| **Users are spread globally** | Latency matters — 200ms vs 20ms is noticeable |
| **Need 99.99%+ availability** | Single region failure shouldn't affect users |
| **Compliance requires data locality** | GDPR: EU data must stay in EU |
| **Disaster recovery requirements** | "If US-East is nuked, service continues" |
| **Massive scale (millions of users)** | One region can't handle all traffic |

### ❌ Don't Use When:

| Criteria | Why |
|----------|-----|
| **All users are in one country** | One region is sufficient |
| **Budget is limited** | Multi-region costs 2-4x (duplicate everything!) |
| **Team is small** | Operational complexity is enormous |
| **Strong consistency is critical everywhere** | Multi-region consistency is extremely hard |
| **< 100,000 users** | Overkill for this scale |

---

## Cost Comparison

```
SINGLE REGION:                    MULTI-REGION (3 regions):
├── 5 app servers:    $150       ├── 15 app servers:   $450
├── 1 database:       $200       ├── 3 databases:      $600
├── 1 Redis:          $50        ├── 3 Redis:          $150
├── Load balancer:    $30        ├── 3 load balancers: $90
├── Data transfer:    $50        ├── Cross-region data: $200
│                                ├── Route 53 health:  $20
├── TOTAL:           ~$480/mo    ├── TOTAL:          ~$1,510/mo
│                                │
│                                │  3x MORE expensive!
│                                │  But: 99.99% uptime vs 99.9%
│                                │       Downtime: 4 min/month vs 43 min/month
│                                │       Global latency: 20ms vs 200ms (for distant users)
```

---

## Key Takeaways

- **Multi-region deployment** means running your application in multiple geographic locations simultaneously for lower latency and higher availability.
- **GeoDNS** routes users to their nearest region — a user in India goes to Mumbai, a user in Europe goes to Ireland.
- **Data replication is the hardest challenge** — you must choose between strong consistency (slow writes) or eventual consistency (fast but stale reads).
- **Active-Active** (all regions serve traffic) is ideal but complex; **Active-Passive** (one primary, others standby) is simpler but wasteful.
- **Failover must be tested** — just having multi-region isn't enough if you've never verified it works.
- **Cost is 2-4x** a single region — you're duplicating everything. Only do this when you have global users or need extreme availability.
- **Netflix, Google, and Amazon all operate multi-region** — it's what enables their 99.99%+ uptime guarantees.

---

## What's Next?

You now understand WHERE to deploy (single server → separate servers → multiple instances → multiple regions). But HOW do you release new code without breaking things? That's **Chapter 5.5: Blue-Green & Canary Deployments — Zero Downtime Releases**, where we explore deployment strategies that let you ship new versions with zero user impact.
