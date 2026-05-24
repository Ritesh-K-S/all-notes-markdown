# Stage 5: DB Replication + Read Replicas — 100K to 1M Users

> **What you'll learn**: How to scale your database by creating copies (replicas) that handle read queries, allowing your system to serve millions of read requests while the primary database focuses on writes.

---

## Real-Life Analogy

Your chai shop now has **one master recipe book** (the primary database). All changes to recipes go here — new drinks, price changes, ingredient updates. But you have 20 chai makers who constantly need to READ the recipes.

If all 20 crowd around one book, it's chaos. 

**Solution**: You make **photocopy duplicates** of the recipe book. Each chai maker gets their own copy. When the master book is updated, the copies are automatically updated too (with a tiny delay). Now 20 people can read simultaneously without blocking each other or the person updating the master.

That's **database replication**. One master (primary) handles writes. Multiple copies (replicas) handle reads.

---

## Core Concept Explained Step-by-Step

### The Architecture

```
                         Load Balancer
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  App Server  │ │  App Server  │ │  App Server  │
     │  Instance 1  │ │  Instance 2  │ │  Instance 3  │
     └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
            │                │                │
            │  (smart routing: writes vs reads)
            │                │                │
            ▼                ▼                ▼
    ┌───────────────────────────────────────────────┐
    │              APPLICATION LAYER                 │
    │                                               │
    │   WRITE operations ──▶  PRIMARY (Master)      │
    │   READ operations  ──▶  REPLICAS (Slaves)     │
    │                                               │
    └───────────────────────────────────────────────┘
            │                              │
            ▼                              ▼
┌────────────────────┐      ┌──────────────────────────────┐
│   PRIMARY (Master) │      │        READ REPLICAS         │
│                    │      │                              │
│  ┌──────────────┐  │      │  ┌──────────┐ ┌──────────┐  │
│  │ PostgreSQL   │  │─────▶│  │ Replica  │ │ Replica  │  │
│  │              │  │ WAL  │  │    1     │ │    2     │  │
│  │ Handles ALL  │  │stream│  │          │ │          │  │
│  │ WRITES       │  │      │  │ Handles  │ │ Handles  │  │
│  │              │  │      │  │ READS    │ │ READS    │  │
│  └──────────────┘  │      │  └──────────┘ └──────────┘  │
│                    │      │                              │
│  INSERT, UPDATE,   │      │  SELECT queries only         │
│  DELETE            │      │                              │
└────────────────────┘      └──────────────────────────────┘
```

### Why This Works

Most applications are **read-heavy**:

```
Typical Web Application Traffic:

████████████████████████████████████████████████░░░░░░ 
▲ 80-95% READS (SELECT)                        ▲ 5-20% WRITES (INSERT/UPDATE/DELETE)

Examples:
- Twitter: 99% reads (viewing tweets) vs 1% writes (posting tweets)
- E-commerce: 95% reads (browsing products) vs 5% writes (placing orders)
- Social media: 90% reads (scrolling feed) vs 10% writes (posting/commenting)

If you can offload reads to replicas:
- Primary handles: 5-20% of total queries (writes only)
- Each replica handles: share of 80-95% of queries
- Result: Effectively 3-5x MORE database capacity
```

### How Replication Works

```
Step-by-step data flow:

1. App writes to PRIMARY:
   INSERT INTO users (name) VALUES ('Alice');

2. PRIMARY writes to local disk AND writes to WAL (Write-Ahead Log):
   ┌───────────────────────────────────┐
   │ WAL (Write-Ahead Log)             │
   │                                   │
   │ LSN 1001: INSERT users 'Alice'    │
   │ LSN 1002: UPDATE orders SET ...   │
   │ LSN 1003: DELETE sessions WHERE...│
   └───────────────────────────────────┘

3. WAL is streamed to replicas in real-time:
   PRIMARY ════WAL Stream════▶ Replica 1
                             ════▶ Replica 2

4. Replicas apply the changes to their local copy:
   Replica 1: "Got it! Applied INSERT for Alice" ✓
   Replica 2: "Got it! Applied INSERT for Alice" ✓

5. Next read query for 'Alice' hits a replica:
   SELECT * FROM users WHERE name = 'Alice';  → Found! ✓
```

### Replication Lag: The Tradeoff

```
                    TIME →
PRIMARY:    [Write Alice] ────────────────────────────────
                    │
                    │ ~10-100ms delay (replication lag)
                    ▼
REPLICA 1:  ─────────────── [Alice appears here] ────────
REPLICA 2:  ─────────────────── [Alice appears here] ────

The PROBLEM:
┌─────────────────────────────────────────────────────────┐
│ User: "I just updated my profile!"                       │
│ App: Writes to PRIMARY ✓                                 │
│ User: *refreshes page*                                   │
│ App: Reads from REPLICA (hasn't received update yet!)    │
│ User: "Where's my update?! The site is broken!"  😡      │
│                                                         │
│ This is called "read-your-writes consistency" problem    │
└─────────────────────────────────────────────────────────┘

SOLUTION: Read from PRIMARY for "own data" after writes
┌─────────────────────────────────────────────────────────┐
│ User updates profile → Write to PRIMARY                  │
│ User views OWN profile → Read from PRIMARY (fresh!)      │
│ Other users view this profile → Read from REPLICA (OK!)  │
└─────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### PostgreSQL Streaming Replication

```
┌────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Replication                            │
│                                                                    │
│  PRIMARY                                                           │
│  ┌─────────────────────────────────┐                               │
│  │ Transaction committed           │                               │
│  │         │                       │                               │
│  │         ▼                       │                               │
│  │ Write to WAL (pg_wal/)          │                               │
│  │         │                       │                               │
│  │         ├──▶ WAL Sender Process ├──── TCP ────▶ Replica 1       │
│  │         │                       │                               │
│  │         └──▶ WAL Sender Process ├──── TCP ────▶ Replica 2       │
│  └─────────────────────────────────┘                               │
│                                                                    │
│  REPLICA                                                           │
│  ┌─────────────────────────────────┐                               │
│  │ WAL Receiver Process            │                               │
│  │         │                       │                               │
│  │         ▼                       │                               │
│  │ Write WAL to local disk         │                               │
│  │         │                       │                               │
│  │         ▼                       │                               │
│  │ Recovery Process applies changes│                               │
│  │         │                       │                               │
│  │         ▼                       │                               │
│  │ Data visible in queries         │                               │
│  └─────────────────────────────────┘                               │
│                                                                    │
│  Modes:                                                            │
│  • Async (default): Primary doesn't wait for replicas (fastest)    │
│  • Sync: Primary waits for at least 1 replica to confirm (safest)  │
└────────────────────────────────────────────────────────────────────┘
```

### Synchronous vs Asynchronous Replication

```
ASYNCHRONOUS (default, faster):
┌────────┐    ┌─────────┐    ┌─────────┐
│ Client │───▶│ PRIMARY │───▶│ REPLICA │
│        │    │         │    │         │
│  WRITE │    │ Commits │    │ Applies │
│        │◀───│ Returns │    │ later   │
│  ACK!  │    │ SUCCESS │    │ (10ms?) │
└────────┘    └─────────┘    └─────────┘
Risk: If PRIMARY crashes BEFORE replica gets the data → data lost!

SYNCHRONOUS (safer, slower):
┌────────┐    ┌─────────┐    ┌─────────┐
│ Client │───▶│ PRIMARY │───▶│ REPLICA │
│        │    │         │    │         │
│  WRITE │    │ Writes  │    │ Receives│
│        │    │ WAL     │    │ Confirms│
│        │    │ Waits...│◀───│ "Got it"│
│        │◀───│ Returns │    │         │
│  ACK!  │    │ SUCCESS │    │         │
└────────┘    └─────────┘    └─────────┘
Guarantee: Data exists on at least 2 machines before client gets ACK
Trade-off: Each write takes 1-5ms longer (network round-trip to replica)
```

### Automatic Failover

```
Normal operation:
┌──────────┐     ┌──────────┐     ┌──────────┐
│ PRIMARY  │────▶│ Replica 1│     │ Replica 2│
│  (active)│     │  (standby)     │  (standby)
└──────────┘     └──────────┘     └──────────┘

PRIMARY crashes! 💥
┌──────────┐     ┌──────────┐     ┌──────────┐
│ PRIMARY  │ ✗   │ Replica 1│     │ Replica 2│
│  (DEAD)  │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘
                       │
            Failover manager detects failure
            (Patroni, pg_auto_failover, AWS RDS)
                       │
                       ▼
                 PROMOTE Replica 1 to PRIMARY!
                       │
┌──────────┐     ┌──────────┐     ┌──────────┐
│ OLD PRIMARY│   │ NEW      │     │ Replica 2│
│  (DEAD)  │     │ PRIMARY  │────▶│ (follows │
│          │     │ (promoted)     │  new primary)
└──────────┘     └──────────┘     └──────────┘

App connection string updates automatically (DNS or proxy)
Downtime: 10-30 seconds (with good tooling)
```

---

## Code Examples

### Python: Read/Write Splitting with SQLAlchemy

```python
# db.py — Route reads to replicas, writes to primary
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import random

# Primary: handles ALL writes
primary_engine = create_engine(
    "postgresql://user:pass@primary.db.internal:5432/myapp",
    pool_size=10,
    max_overflow=20
)

# Replicas: handle reads (load-balance between them)
replica_engines = [
    create_engine(
        "postgresql://user:pass@replica1.db.internal:5432/myapp",
        pool_size=10
    ),
    create_engine(
        "postgresql://user:pass@replica2.db.internal:5432/myapp",
        pool_size=10
    ),
]

PrimarySession = sessionmaker(bind=primary_engine)
ReplicaSessions = [sessionmaker(bind=engine) for engine in replica_engines]

def get_write_session():
    """Use for INSERT, UPDATE, DELETE"""
    return PrimarySession()

def get_read_session():
    """Use for SELECT — randomly picks a replica"""
    session_class = random.choice(ReplicaSessions)
    return session_class()

# Usage in API endpoints:
@app.route("/api/users/<int:user_id>")
def get_user(user_id):
    session = get_read_session()  # Goes to a REPLICA
    try:
        user = session.query(User).filter_by(id=user_id).first()
        return jsonify(user.to_dict())
    finally:
        session.close()

@app.route("/api/users", methods=["POST"])
def create_user():
    session = get_write_session()  # Goes to PRIMARY
    try:
        user = User(name=request.json["name"], email=request.json["email"])
        session.add(user)
        session.commit()
        return jsonify(user.to_dict()), 201
    finally:
        session.close()

@app.route("/api/users/<int:user_id>/profile")
def get_own_profile(user_id):
    """Read-your-writes: user views own profile after edit → use PRIMARY"""
    just_updated = request.args.get("fresh") == "true"
    if just_updated:
        session = get_write_session()  # Read from PRIMARY (guaranteed fresh)
    else:
        session = get_read_session()   # Normal reads from replica
    try:
        user = session.query(User).filter_by(id=user_id).first()
        return jsonify(user.to_dict())
    finally:
        session.close()
```

### Java: Read/Write Splitting with Spring Boot

```java
// DataSourceConfig.java — Multiple datasources for read/write splitting
@Configuration
public class DataSourceConfig {

    @Bean("primaryDataSource")
    @Primary
    public DataSource primaryDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://primary.db.internal:5432/myapp");
        config.setUsername("appuser");
        config.setPassword("secret");
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }

    @Bean("replicaDataSource")
    public DataSource replicaDataSource() {
        HikariConfig config = new HikariConfig();
        // Multiple replicas via pgBouncer or connection proxy
        config.setJdbcUrl("jdbc:postgresql://replicas.db.internal:5432/myapp");
        config.setUsername("appuser");
        config.setPassword("secret");
        config.setMaximumPoolSize(30);
        config.setReadOnly(true);  // Safety: prevent accidental writes
        return new HikariDataSource(config);
    }

    @Bean
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {
        ReadWriteRoutingDataSource router = new ReadWriteRoutingDataSource();
        router.setPrimary(primary);
        router.setReplica(replica);
        return router;
    }
}

// ReadWriteRoutingDataSource.java — Smart routing
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {

    private static final ThreadLocal<Boolean> IS_READ_ONLY = new ThreadLocal<>();

    public static void setReadOnly(boolean readOnly) {
        IS_READ_ONLY.set(readOnly);
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return Boolean.TRUE.equals(IS_READ_ONLY.get()) ? "replica" : "primary";
    }
}

// UserService.java — Annotate methods for routing
@Service
public class UserService {

    @Transactional(readOnly = true)  // Routes to REPLICA
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @Transactional  // Routes to PRIMARY (readOnly=false is default)
    public User createUser(CreateUserRequest request) {
        User user = new User(request.getName(), request.getEmail());
        return userRepository.save(user);
    }
}
```

---

## Infrastructure Example

### PostgreSQL Streaming Replication Setup

```bash
# On PRIMARY server (postgresql.conf):
wal_level = replica
max_wal_senders = 5           # Max number of replicas
wal_keep_size = 1GB           # Keep WAL for slow replicas
synchronous_commit = on       # Wait for at least 1 replica

# Create replication user:
# CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'repl_pass';

# pg_hba.conf — Allow replicas to connect:
# host replication replicator 10.0.0.0/16 md5
```

```bash
# On REPLICA server — Initial setup:
# Step 1: Copy all data from primary
pg_basebackup -h primary.db.internal -U replicator -D /var/lib/postgresql/data -P -R

# Step 2: The -R flag creates standby.signal and sets:
# primary_conninfo in postgresql.auto.conf

# Step 3: Start the replica
pg_ctl start

# The replica is now streaming WAL from the primary!
```

### AWS RDS Read Replica (Terraform)

```hcl
# rds.tf — Primary + Read Replicas on AWS

# Primary database
resource "aws_db_instance" "primary" {
  identifier     = "myapp-primary"
  engine         = "postgres"
  engine_version = "15"
  instance_class = "db.r6g.xlarge"   # 4 vCPU, 32 GB RAM
  allocated_storage = 500

  db_name  = "myapp"
  username = "appuser"
  password = var.db_password

  # Enable automated backups (required for replicas)
  backup_retention_period = 7

  # Multi-AZ for automatic failover
  multi_az = true

  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.private.name
}

# Read Replica 1
resource "aws_db_instance" "replica1" {
  identifier          = "myapp-replica-1"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class      = "db.r6g.xlarge"

  # Replica can be in a different AZ for resilience
  availability_zone = "us-east-1b"
}

# Read Replica 2
resource "aws_db_instance" "replica2" {
  identifier          = "myapp-replica-2"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class      = "db.r6g.xlarge"

  availability_zone = "us-east-1c"
}

# Output endpoints for app configuration
output "primary_endpoint" {
  value = aws_db_instance.primary.endpoint
}
output "replica_endpoints" {
  value = [
    aws_db_instance.replica1.endpoint,
    aws_db_instance.replica2.endpoint
  ]
}
```

### Monitoring Replication Lag

```sql
-- On the PRIMARY: check replication status
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    extract(epoch from now() - reply_time) AS lag_seconds
FROM pg_stat_replication;

-- On the REPLICA: check how far behind
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- Alert if lag > 5 seconds (production threshold)
```

---

## Real-World Example

### YouTube's Read Replica Strategy

YouTube serves **500 hours of video uploaded every minute** and billions of views:
- Video metadata (title, description, view count) is read millions of times per second
- They use dozens of read replicas across regions
- View counts are eventually consistent (you might see "1.2M views" that's 30 seconds stale — that's fine!)

### Shopify During Flash Sales

When a popular brand launches a sale on Shopify:
- Product page views increase 1000x in seconds
- Read replicas handle the massive read spike
- Only actual purchases (writes) hit the primary
- Without replicas, the database would collapse under read load

### Facebook's MySQL Replication

Facebook runs one of the world's largest MySQL deployments:
- Thousands of read replicas across data centers
- Cross-region replication with 50-100ms lag (acceptable for social feeds)
- Users see their own posts immediately (read-your-writes from primary)
- Other users see posts after replication lag (usually < 1 second)

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Reading from replica immediately after write** | User doesn't see their own changes | Read from primary for "own data" right after writes |
| **Not monitoring replication lag** | Replicas fall behind without anyone noticing | Alert when lag > 1-5 seconds |
| **Too many connections to primary** | Writes still bottlenecked | Use pgBouncer for connection pooling |
| **Treating replicas as backup** | Replication ≠ backup (if you corrupt primary, replicas get corrupted too) | Keep separate point-in-time backups |
| **Not testing failover** | Failover is untested → fails when you need it most | Practice failover regularly (chaos engineering) |
| **Unbalanced replica load** | One replica gets 90% of traffic | Use load balancer across replicas |
| **Replication lag during bulk operations** | Large migrations cause replicas to fall hours behind | Throttle bulk operations, or do them during low traffic |

---

## When to Use / When NOT to Use

### ✅ Add Read Replicas When:
- Database CPU is high and most queries are SELECTs
- You need to handle 10x more read traffic
- You want automatic failover (high availability)
- You can tolerate slight staleness (seconds) in read data
- Analytics/reporting queries are slowing down the primary

### ❌ Don't Add Replicas When:
- Your workload is write-heavy (replicas don't help with writes)
- You can't handle ANY data staleness (financial transactions)
- Adding replicas won't fix the problem (e.g., one slow query on primary)
- You haven't optimized your queries and indexes first
- The bottleneck is a specific write-heavy table (need sharding instead)

---

## Key Takeaways

1. **Most apps are 80-95% reads** — offloading reads to replicas dramatically reduces primary load
2. **Replication lag is the key tradeoff** — reads from replicas may be milliseconds behind
3. **Read-your-writes pattern** — always read user's own recent changes from primary
4. **Automatic failover provides high availability** — if primary dies, a replica takes over in seconds
5. **Replicas DON'T help with write scalability** — all writes still go to one primary
6. **Monitor replication lag religiously** — it's your canary in the coal mine
7. **AWS RDS makes this trivially easy** — create a read replica with one API call

---

## What's Next?

Read replicas solve the database read bottleneck, but what about when your application logic becomes too complex for a monolith? When different teams need to deploy independently? When different features need different scaling strategies?

That's when you move to **Microservices + Message Queues** — Stage 6.

Next: [06-stage6-microservices.md](./06-stage6-microservices.md)
