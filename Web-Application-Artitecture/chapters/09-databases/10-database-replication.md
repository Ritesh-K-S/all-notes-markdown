# Database Replication — Master-Slave, Master-Master

> **What you'll learn**: How databases copy data across multiple servers for high availability and read scaling, the difference between synchronous and asynchronous replication, master-slave vs master-master topologies, and how to handle replication lag.

---

## Real-Life Analogy

Imagine a popular **restaurant** with one chef (the master database). During peak hours, orders pile up and customers wait forever.

**Solution: Replication = Training clone chefs**
- The **head chef** (Primary/Master) creates every new dish (handles writes)
- **Clone chefs** (Replicas/Slaves) watch what the head chef does and replicate every dish exactly
- Customers can get their food from ANY clone chef (read from any replica)
- If the head chef gets sick, one clone takes over (failover)

Now the restaurant serves 5x more customers with the same menu!

---

## Core Concept Explained Step-by-Step

### Why Replicate?

```
WITHOUT REPLICATION:                    WITH REPLICATION:
┌────────────────────┐                 ┌────────────────────┐
│    Single Server   │                 │   Primary (Write)  │
│                    │                 └──────────┬─────────┘
│  • One failure =   │                            │ Replicate
│    TOTAL DOWNTIME  │                  ┌─────────┼─────────┐
│  • All reads hit   │                  ▼         ▼         ▼
│    one server      │           ┌──────────┐┌──────────┐┌──────────┐
│  • No disaster     │           │ Replica 1││ Replica 2││ Replica 3│
│    recovery        │           │ (Read)   ││ (Read)   ││ (Read)   │
│                    │           └──────────┘└──────────┘└──────────┘
└────────────────────┘
                                  ✅ High Availability (failover)
                                  ✅ Read Scaling (4x capacity)
                                  ✅ Disaster Recovery (backup)
                                  ✅ Geographic Distribution
```

### Replication Topologies

```
1. MASTER-SLAVE (Primary-Replica):
───────────────────────────────────
   Writes ──▶ ┌──────────┐
              │  PRIMARY  │──replicates──▶ Replica 1 ──▶ Reads
              │  (Master) │──replicates──▶ Replica 2 ──▶ Reads
   Reads ───▶ │           │──replicates──▶ Replica 3 ──▶ Reads
              └──────────┘
   
   ✅ Simple, well-understood
   ✅ No write conflicts
   ❌ Single point of write failure
   ❌ Replicas may be slightly behind (lag)

2. MASTER-MASTER (Multi-Primary):
───────────────────────────────────
   Writes ──▶ ┌──────────┐ ◀──replicates──▶ ┌──────────┐ ◀── Writes
   Reads ───▶ │ Primary 1│                   │ Primary 2│ ◀── Reads
              └──────────┘                   └──────────┘
   
   ✅ Both nodes accept writes
   ✅ No single point of failure for writes
   ❌ CONFLICT RESOLUTION needed (same row updated on both!)
   ❌ More complex, harder to debug

3. CASCADING REPLICATION:
───────────────────────────────────
   Primary ──▶ Replica A ──▶ Replica B ──▶ Replica C
              (close)       (far)         (very far)
   
   ✅ Reduces load on primary (doesn't replicate to all)
   ❌ More lag for downstream replicas
```

### Synchronous vs Asynchronous Replication

```
SYNCHRONOUS:
─────────────────────────────────────────────────
Client ──▶ Primary: "INSERT user"
                │
                ├──▶ Write to local WAL ✓
                ├──▶ Send to Replica 1... wait...
                │         Replica 1: "Got it! ✓"
                ├──▶ Send to Replica 2... wait...
                │         Replica 2: "Got it! ✓"
                │
                ◀── "OK, write committed!"

✅ Zero data loss (all replicas have the data)
❌ Slower (must wait for replicas)
❌ If replica is slow/down, writes are blocked

ASYNCHRONOUS:
─────────────────────────────────────────────────
Client ──▶ Primary: "INSERT user"
                │
                ├──▶ Write to local WAL ✓
                │
                ◀── "OK, write committed!" (immediate!)
                │
                └──▶ Send to Replicas (background, best-effort)
                     Replica 1: "Got it!" (maybe 10ms later)
                     Replica 2: "Got it!" (maybe 50ms later)

✅ Fast writes (no waiting)
✅ Primary not blocked by slow replicas
❌ Data loss possible if primary crashes before replication
❌ Replicas may serve stale data (replication lag)

SEMI-SYNCHRONOUS (Compromise):
─────────────────────────────────────────────────
Wait for at least ONE replica to confirm, then commit.
✅ At least one copy guaranteed
✅ Faster than full synchronous
```

---

## How It Works Internally

### PostgreSQL Streaming Replication

```
┌─────────────────────────────────────────────────────────────────┐
│                PostgreSQL Replication Architecture                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PRIMARY SERVER:                                                 │
│  ┌─────────────────────────────────────────────┐                │
│  │ Client writes → WAL (Write-Ahead Log)       │                │
│  │                                             │                │
│  │ WAL Segments: [000001] [000002] [000003]    │                │
│  │                    │                         │                │
│  │              WAL Sender Process              │                │
│  │              (streams WAL bytes)             │                │
│  └────────────────────┬────────────────────────┘                │
│                       │ TCP Stream                               │
│                       ▼                                          │
│  REPLICA SERVER:                                                 │
│  ┌─────────────────────────────────────────────┐                │
│  │              WAL Receiver Process            │                │
│  │              (receives WAL bytes)            │                │
│  │                    │                         │                │
│  │                    ▼                         │                │
│  │ Apply WAL → Replay same changes on replica  │                │
│  │                                             │                │
│  │ Data is now identical to primary!            │                │
│  └─────────────────────────────────────────────┘                │
│                                                                   │
│  Physical Replication: Byte-for-byte WAL replay                  │
│  Logical Replication: SQL-level changes (more flexible)         │
└─────────────────────────────────────────────────────────────────┘
```

### MySQL Binary Log Replication

```
PRIMARY (Source):
┌─────────────────────────────────────┐
│ Transaction committed               │
│       │                             │
│       ▼                             │
│ Binary Log (binlog):                │
│ ┌──────────────────────────────┐   │
│ │ Event: INSERT INTO users...  │   │
│ │ Event: UPDATE orders SET...  │   │
│ │ Event: DELETE FROM cart...   │   │
│ └──────────────────────────────┘   │
└───────────────┬─────────────────────┘
                │ Binlog Dump Thread
                ▼
REPLICA:
┌─────────────────────────────────────┐
│ I/O Thread: Receives binlog events  │
│       │                             │
│       ▼                             │
│ Relay Log (local copy of binlog)    │
│       │                             │
│       ▼                             │
│ SQL Thread: Replays events locally  │
│       │                             │
│       ▼                             │
│ Data updated on replica!            │
└─────────────────────────────────────┘
```

### Replication Lag — The Critical Challenge

```
TIME ──────────────────────────────────────────────────▶

Primary:    [Write A] [Write B] [Write C] [Write D]
                │         │         │         │
Replica:    [Write A] [Write B] [   lag   ] [Write C]...
                                     ▲
                                     │
                            REPLICATION LAG
                            (Replica is behind!)

Problem scenario:
1. User updates profile on Primary: name = "Alice Smith"
2. User immediately reads profile (routed to Replica)
3. Replica hasn't received the update yet!
4. User sees OLD name: "Alice Johnson"  ← CONFUSING!

This is called "read-your-writes" inconsistency
```

### Handling Replication Lag

```
STRATEGY 1: Read from Primary for recent writes
─────────────────────────────────────────────────
After write: route reads to primary for N seconds
After N seconds: route back to replica

STRATEGY 2: Causal consistency with timestamps
─────────────────────────────────────────────────
Write returns: {timestamp: "2024-01-15T10:00:05"}
Read request includes: "give me data at least as fresh as 10:00:05"
Replica: "My latest is 10:00:03... wait... now 10:00:05. Here you go!"

STRATEGY 3: Session stickiness
─────────────────────────────────────────────────
Same user session → always reads from same replica
Guarantees they see their own writes (that replica has them)
```

---

## Code Examples

### Python (PostgreSQL Replication-Aware Reads)

```python
import psycopg2
from psycopg2 import pool
import random

# Connection pools for primary and replicas
primary_pool = psycopg2.pool.ThreadedConnectionPool(
    5, 20, host="primary.db.internal", database="myapp",
    user="app_user", password="secret"
)

replica_pools = [
    psycopg2.pool.ThreadedConnectionPool(
        5, 20, host=f"replica{i}.db.internal", database="myapp",
        user="app_readonly", password="secret"
    ) for i in range(1, 4)
]

def write_query(sql, params=None):
    """All writes go to primary"""
    conn = primary_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute(sql, params)
        conn.commit()
        return cursor.fetchone() if cursor.description else None
    finally:
        primary_pool.putconn(conn)

def read_query(sql, params=None, require_fresh=False):
    """Reads go to replicas (or primary if freshness required)"""
    if require_fresh:
        # Critical reads go to primary
        pool = primary_pool
    else:
        # Distribute reads across replicas (round-robin)
        pool = random.choice(replica_pools)
    
    conn = pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute(sql, params)
        return cursor.fetchall()
    finally:
        pool.putconn(conn)

# Usage
# Write: always goes to primary
write_query(
    "UPDATE users SET name = %s WHERE id = %s",
    ("Alice Smith", 1)
)

# Read: goes to replica (stale data OK)
users = read_query("SELECT * FROM users WHERE age > %s", (25,))

# Critical read after write: goes to primary
profile = read_query(
    "SELECT * FROM users WHERE id = %s", (1,),
    require_fresh=True  # Just updated, need fresh data!
)
```

### Java (Spring Boot with Read/Write Splitting)

```java
import javax.sql.DataSource;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

// Custom routing datasource
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        // If current transaction is read-only → use replica
        // Otherwise → use primary
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? "replica" : "primary";
    }
}

// Service layer
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    // Writes go to primary (default)
    @Transactional
    public User updateUser(Long id, String name) {
        User user = userRepository.findById(id).orElseThrow();
        user.setName(name);
        return userRepository.save(user);
    }

    // Reads go to replica (read-only transaction)
    @Transactional(readOnly = true)
    public List<User> findActiveUsers() {
        return userRepository.findByStatus("active");
    }

    // Force read from primary (after write, need fresh data)
    @Transactional(readOnly = false)  // Goes to primary
    public User getUserFresh(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

---

## Infrastructure Examples

### PostgreSQL Streaming Replication Setup

```bash
# On Primary (postgresql.conf):
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB
synchronous_standby_names = 'replica1'

# On Primary (pg_hba.conf) — allow replication connections:
host replication replicator replica1_ip/32 scram-sha-256
```

```bash
# On Replica — create base backup from primary:
pg_basebackup -h primary_ip -D /var/lib/postgresql/data -U replicator -P -R
# -R creates standby.signal and sets primary_conninfo automatically
```

### Docker Compose: PostgreSQL Primary + Replicas

```yaml
version: '3.8'
services:
  postgres-primary:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - ./primary/postgresql.conf:/etc/postgresql/postgresql.conf
      - primary-data:/var/lib/postgresql/data
    command: postgres -c config_file=/etc/postgresql/postgresql.conf

  postgres-replica1:
    image: postgres:16
    environment:
      PGUSER: replicator
      PGPASSWORD: replpass
    depends_on:
      - postgres-primary
    volumes:
      - replica1-data:/var/lib/postgresql/data
    command: |
      bash -c "
        until pg_basebackup -h postgres-primary -D /var/lib/postgresql/data -U replicator -P -R; do
          sleep 1
        done
        postgres
      "

  pgpool:
    image: pgpool/pgpool
    ports:
      - "5433:5432"
    environment:
      - PGPOOL_BACKEND_NODES=0:postgres-primary:5432,1:postgres-replica1:5432
      - PGPOOL_SR_CHECK_USER=admin
      - PGPOOL_ENABLE_LOAD_BALANCING=yes

volumes:
  primary-data:
  replica1-data:
```

### AWS RDS with Read Replicas (Terraform)

```hcl
# Primary instance
resource "aws_db_instance" "primary" {
  identifier     = "myapp-primary"
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = "db.r6g.xlarge"
  
  multi_az               = true   # Synchronous standby for HA
  backup_retention_period = 7
  
  db_name  = "myapp"
  username = "admin"
  password = var.db_password
}

# Read replicas (asynchronous)
resource "aws_db_instance" "replica" {
  count = 3
  
  identifier          = "myapp-replica-${count.index + 1}"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class      = "db.r6g.large"
  
  # Can be in different AZs or even different regions!
  availability_zone = element(var.azs, count.index)
}
```

---

## Real-World Example

### GitHub — PostgreSQL Replication at Scale

```
┌─────────────────────────────────────────────────────────────────┐
│              GitHub's Database Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────┐                                         │
│  │   Primary (Write)  │ ← All writes (pushes, PRs, issues)     │
│  │   MySQL 8.0        │                                         │
│  └────────┬───────────┘                                         │
│           │                                                      │
│     Replication (semi-sync)                                      │
│           │                                                      │
│  ┌────────┴───────────────────────────────────────┐             │
│  │                                                 │             │
│  ▼                    ▼                    ▼       │             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │             │
│  │Replica 1 │  │Replica 2 │  │Replica 3 │       │             │
│  │(Online   │  │(Online   │  │(Search   │       │             │
│  │ reads)   │  │ reads)   │  │ queries) │  ...more             │
│  └──────────┘  └──────────┘  └──────────┘                      │
│                                                                   │
│  ProxySQL: Routes reads to replicas, writes to primary          │
│  Orchestrator: Automated failover (promotes replica to primary)  │
│                                                                   │
│  Key Stats:                                                      │
│  • 1200+ MySQL hosts                                             │
│  • Multiple replicas per primary                                 │
│  • Cross-datacenter replication for DR                           │
│  • Sub-second failover with Orchestrator                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Reading from replica immediately after write | See stale data (replication lag) | Read from primary for N seconds after write |
| No monitoring of replication lag | Don't know when replicas fall behind | Alert when lag > threshold (e.g., 5 seconds) |
| Synchronous replication to distant replicas | Massive write latency | Use sync only to local standby, async for distant |
| No automatic failover | Manual promotion during outage = downtime | Use Patroni (PG), Orchestrator (MySQL), or managed DB |
| Too many replicas from one primary | Primary overwhelmed by replication connections | Use cascading replication |
| Not testing failover | Failover breaks in production | Regular failover drills (chaos engineering) |
| Ignoring split-brain scenarios | Both nodes think they're primary → data corruption | Use fencing (STONITH) or consensus (Patroni + etcd) |

---

## When to Use / When NOT to Use

### ✅ Use Replication When:
- **High availability** — application can't tolerate downtime
- **Read-heavy workloads** — 80%+ reads, need to scale reads
- **Disaster recovery** — need a copy in another region/datacenter
- **Reporting/analytics** — run heavy queries without impacting production
- **Geographic distribution** — replicas close to users for lower latency

### ❌ Replication Won't Help When:
- **Write-heavy workloads** — all writes still go to one primary
- **Need strong consistency everywhere** — replicas have lag
- **Small databases** — single server handles the load fine
- **Need to scale writes** — use sharding instead (Chapter 9.11)

---

## Key Takeaways

1. **Replication** copies data from a primary to one or more replicas — enabling high availability, read scaling, and disaster recovery.
2. **Asynchronous replication** is fast but risks data loss; **synchronous** is safe but adds latency. Most production systems use semi-synchronous.
3. **Read/write splitting** routes writes to primary and reads to replicas — the most common pattern for scaling read-heavy workloads.
4. **Replication lag** is inevitable with async replication — design your application to handle it (read-your-writes, causal consistency).
5. **Automatic failover** (Patroni, Orchestrator, managed services) is essential — manual failover in the middle of an outage is stressful and slow.
6. **Master-master replication** sounds appealing but introduces write conflicts — use it only when you truly need multi-region writes.
7. **Replication scales reads, not writes** — if you need to scale writes, you need sharding (splitting data across multiple primaries).

---

## What's Next?

Next, we'll explore **Database Sharding — Splitting Data Across Machines** (Chapter 9.11), where you'll learn how to distribute data across multiple database servers when a single primary can't handle the write load.
