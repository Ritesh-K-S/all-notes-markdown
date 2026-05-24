# NewSQL Databases (CockroachDB, Google Spanner, TiDB)

> **What you'll learn**: How NewSQL databases solve the impossible trade-off between SQL's ACID guarantees and NoSQL's horizontal scalability, how Google Spanner uses atomic clocks for global consistency, how CockroachDB achieves distributed SQL without special hardware, and when NewSQL is the right choice for planet-scale applications.

---

## Real-Life Analogy

Imagine running a **global banking system**:

**Traditional SQL** (PostgreSQL): One super-powerful vault in one city. Perfect record-keeping (ACID), but everyone worldwide must travel to that ONE vault. Gets overwhelmed at scale. Can't survive if that city has an earthquake.

**NoSQL** (Cassandra): Vaults in every city, incredibly fast deposits. But... sometimes two cities disagree on your balance. "Eventually" they'll agree. Not great for banking!

**NewSQL** (Spanner/CockroachDB): Vaults in every city, all perfectly synchronized using **atomic clocks** (Spanner) or **logical clocks** (CockroachDB). You get:
- Speed of local access (vault in your city)
- Perfect ACID guarantees (balances always correct)
- Survives entire cities going offline

**The promise**: Distributed like NoSQL, consistent like SQL. No compromises.

---

## Core Concept Explained Step-by-Step

### The Problem NewSQL Solves

```
THE DATABASE TRILEMMA (before NewSQL):
──────────────────────────────────────────────────────────────────────

Choose 2 out of 3:

         SQL Features (ACID, JOINs, schemas)
              ╱                   ╲
             ╱                     ╲
    Traditional SQL            ???  ← NewSQL fills this gap!
    (PostgreSQL)               (ACID + Scale + SQL)
             ╲                     ╱
              ╲                   ╱
    Horizontal Scalability────NoSQL
    (add more servers)        (Cassandra, DynamoDB)


Traditional SQL:  ✅ ACID  ✅ SQL/JOINs  ❌ Horizontal Scale
NoSQL:            ❌ ACID* ❌ SQL/JOINs  ✅ Horizontal Scale
NewSQL:           ✅ ACID  ✅ SQL/JOINs  ✅ Horizontal Scale  ← ALL THREE!

* Some NoSQL offers limited ACID (single-partition transactions)
```

### How NewSQL Achieves the "Impossible"

```
KEY INNOVATIONS:
──────────────────────────────────────────────────────────────────────

1. DISTRIBUTED CONSENSUS (Raft/Paxos):
   Every write is agreed upon by a MAJORITY of nodes
   → Even if nodes crash, committed data is never lost
   → Unlike primary-replica where replica lag causes stale reads

2. DISTRIBUTED TRANSACTIONS:
   2-Phase Commit + Consensus = ACID across multiple machines
   → Write to table A on Node 1 AND table B on Node 3 atomically
   → Traditional SQL can't do this across servers!

3. AUTOMATIC SHARDING:
   Data automatically split across nodes (ranges of keys)
   → No manual shard configuration
   → Automatic rebalancing when nodes added/removed

4. GLOBAL ORDERING (Spanner's breakthrough):
   Know which transaction happened "first" across the GLOBE
   → GPS + atomic clocks provide globally-synchronized time
   → CockroachDB achieves similar with Hybrid Logical Clocks (HLC)
```

---

## Google Spanner — The Pioneer

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GOOGLE SPANNER ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                      TrueTime API                                ││
│  │  GPS receivers + Atomic Clocks in EVERY data center             ││
│  │  Returns: [earliest_possible_time, latest_possible_time]        ││
│  │  Uncertainty window: typically 1-7 milliseconds                  ││
│  │                                                                   ││
│  │  Purpose: "I know for CERTAIN this event happened after         ││
│  │           that event" — even across continents!                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Zone US-East    │  │  Zone Europe     │  │  Zone Asia       │  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │  │
│  │  │ Span Server│  │  │  │ Span Server│  │  │  │ Span Server│  │  │
│  │  │  Tablet A  │  │  │  │  Tablet A  │  │  │  │  Tablet A  │  │  │
│  │  │  (replica) │  │  │  │  (LEADER)  │  │  │  │  (replica) │  │  │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │  │
│  │  │ Span Server│  │  │  │ Span Server│  │  │  │ Span Server│  │  │
│  │  │  Tablet B  │  │  │  │  Tablet B  │  │  │  │  Tablet B  │  │  │
│  │  │  (LEADER)  │  │  │  │  (replica) │  │  │  │  (replica) │  │  │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                       │
│  HOW TRUETIME ENABLES GLOBAL CONSISTENCY:                            │
│  ─────────────────────────────────────────                          │
│  Transaction T1 commits at TrueTime = [100ms, 107ms]                │
│  Spanner WAITS until 107ms has definitely passed (commit-wait)      │
│  → Any subsequent transaction T2 will have timestamp > 107ms       │
│  → GLOBALLY ORDERED: T2 is guaranteed to see T1's writes           │
│  → Cost: ~7ms extra latency per write (the uncertainty window)     │
└─────────────────────────────────────────────────────────────────────┘
```

### CockroachDB — Spanner Without Atomic Clocks

```
┌─────────────────────────────────────────────────────────────────────┐
│                  COCKROACHDB ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  No atomic clocks! Uses Hybrid Logical Clocks (HLC) instead:       │
│                                                                       │
│  HLC = max(wall_clock_time, last_seen_timestamp) + logical_counter │
│                                                                       │
│  ┌─────────┐       ┌─────────┐       ┌─────────┐                   │
│  │ Node 1  │◀─────▶│ Node 2  │◀─────▶│ Node 3  │                   │
│  │ (NYC)   │ Raft  │ (London)│ Raft  │ (Mumbai)│                   │
│  │         │ Gossip│         │ Gossip│         │                   │
│  └────┬────┘       └────┬────┘       └────┬────┘                   │
│       │                  │                  │                        │
│  ┌────▼────┐       ┌────▼────┐       ┌────▼────┐                   │
│  │ Ranges  │       │ Ranges  │       │ Ranges  │                   │
│  │ [A-G]   │       │ [H-N]   │       │ [O-Z]   │                   │
│  │ Leader  │       │ Leader  │       │ Leader  │                   │
│  │ for some│       │ for some│       │ for some│                   │
│  │ ranges  │       │ ranges  │       │ ranges  │                   │
│  └─────────┘       └─────────┘       └─────────┘                   │
│                                                                       │
│  KEY CONCEPTS:                                                       │
│  • Data split into "ranges" (64MB chunks)                           │
│  • Each range replicated 3x across nodes (Raft consensus)          │
│  • Range leader handles writes (elected by Raft)                    │
│  • Automatic range splitting when data grows                        │
│  • Automatic rebalancing when nodes join/leave                      │
│                                                                       │
│  DISTRIBUTED TRANSACTION (2PC + Raft):                               │
│  1. Client → Gateway node (any node)                                │
│  2. Gateway identifies which ranges are involved                    │
│  3. Writes intents (locks) to each range's Raft leader             │
│  4. If ALL succeed → commit (Parallel Commits optimization)        │
│  5. If ANY fail → abort, clean up intents                          │
└─────────────────────────────────────────────────────────────────────┘
```

### TiDB — MySQL-Compatible NewSQL

```
┌─────────────────────────────────────────────────────────────────────┐
│                      TiDB ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  TiDB = "Ti" (Titanium) + "DB"                                      │
│  MySQL wire protocol compatible → drop-in replacement!              │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TiDB Server (Stateless SQL Layer)                           │   │
│  │  • Parses SQL                                                │   │
│  │  • Optimizes queries                                         │   │
│  │  • Coordinates distributed execution                         │   │
│  │  • Horizontally scalable (just add more TiDB nodes)         │   │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │                                      │
│  ┌────────────────────────────▼────────────────────────────────┐   │
│  │  Placement Driver (PD) — The "Brain"                         │   │
│  │  • Stores metadata (which data is where)                     │   │
│  │  • Generates globally unique timestamps (TSO)               │   │
│  │  • Schedules data movement (rebalancing)                    │   │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │                                      │
│  ┌────────────────────────────▼────────────────────────────────┐   │
│  │  TiKV (Distributed KV Storage Layer)                         │   │
│  │  • RocksDB per node (LSM-Tree storage engine)               │   │
│  │  • Data in "Regions" (96MB chunks), replicated 3x via Raft  │   │
│  │  • MVCC for snapshot isolation                               │   │
│  │  • Percolator-based distributed transactions                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  TiFlash (Columnar Analytics Engine) — HTAP!                 │   │
│  │  • Column-store replica of TiKV data                        │   │
│  │  • Real-time analytics without ETL                          │   │
│  │  • Same SQL, same transaction, different storage layout     │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Comparison Table

```
┌──────────────────────────────────────────────────────────────────────────┐
│              NewSQL DATABASE COMPARISON                                    │
├───────────────┬──────────────┬──────────────────┬────────────────────────┤
│ Feature       │ Spanner      │ CockroachDB      │ TiDB                   │
├───────────────┼──────────────┼──────────────────┼────────────────────────┤
│ Company       │ Google       │ Cockroach Labs   │ PingCAP                │
│ Managed Cloud │ Cloud Spanner│ CockroachDB Cloud│ TiDB Cloud             │
│ Open Source   │ No           │ Yes (BSL)        │ Yes (Apache 2.0)       │
│ SQL Dialect   │ GoogleSQL    │ PostgreSQL       │ MySQL                  │
│ Consistency   │ External     │ Serializable     │ Snapshot Isolation     │
│ Clock Type    │ TrueTime(HW) │ HLC (software)   │ TSO (centralized)      │
│ Min Nodes     │ 3 (managed)  │ 3                │ 3 TiKV + 3 PD + n TiDB│
│ Geo-Distrib.  │ Native       │ Native           │ Planned/Limited        │
│ HTAP          │ Limited      │ No               │ Yes (TiFlash)          │
│ Write Latency │ ~7ms (global)│ ~10-50ms (global)│ ~10ms (single region)  │
│ Use Case      │ Google-scale │ Global OLTP      │ MySQL replacement      │
│               │ banking      │ multi-region     │ with scale             │
└───────────────┴──────────────┴──────────────────┴────────────────────────┘
```

---

## Code Examples

### Python — CockroachDB (PostgreSQL-compatible)

```python
import psycopg2
from psycopg2 import errors
import time

# CockroachDB uses standard PostgreSQL protocol!
conn = psycopg2.connect(
    "postgresql://admin@cockroach-node1:26257/myapp?sslmode=require"
)

def create_schema():
    """CockroachDB-optimized schema"""
    cursor = conn.cursor()
    cursor.execute("""
        -- UUID primary keys (avoid hotspots from sequential IDs)
        CREATE TABLE IF NOT EXISTS accounts (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            owner STRING NOT NULL,
            balance DECIMAL NOT NULL DEFAULT 0,
            region STRING NOT NULL,
            created_at TIMESTAMPTZ DEFAULT now(),
            
            -- Constraint: balance can never go negative
            CONSTRAINT balance_non_negative CHECK (balance >= 0)
        );
        
        -- Geo-partition: keep data close to users
        ALTER TABLE accounts PARTITION BY LIST (region) (
            PARTITION us_east VALUES IN ('us-east-1', 'us-east-2'),
            PARTITION eu_west VALUES IN ('eu-west-1', 'eu-west-2'),
            PARTITION ap_south VALUES IN ('ap-south-1', 'ap-south-2')
        );
        
        -- Pin partitions to specific regions (locality-aware)
        ALTER PARTITION us_east OF TABLE accounts
            CONFIGURE ZONE USING constraints='[+region=us-east]';
        ALTER PARTITION eu_west OF TABLE accounts
            CONFIGURE ZONE USING constraints='[+region=eu-west]';
    """)
    conn.commit()

def transfer_money(from_id, to_id, amount, max_retries=3):
    """
    Distributed ACID transaction across potentially different nodes.
    CockroachDB may retry serialization conflicts — handle them!
    """
    for attempt in range(max_retries):
        try:
            cursor = conn.cursor()
            
            # SERIALIZABLE isolation (CockroachDB default — strongest!)
            cursor.execute("BEGIN")
            
            # Read both balances (locks acquired via MVCC)
            cursor.execute(
                "SELECT id, balance FROM accounts WHERE id IN (%s, %s)",
                (from_id, to_id)
            )
            rows = {row[0]: row[1] for row in cursor.fetchall()}
            
            if rows[from_id] < amount:
                cursor.execute("ROLLBACK")
                raise ValueError("Insufficient balance")
            
            # Transfer (these might be on DIFFERENT nodes — still ACID!)
            cursor.execute(
                "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_id)
            )
            cursor.execute(
                "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_id)
            )
            
            cursor.execute("COMMIT")
            print(f"Transfer complete: {amount} from {from_id} to {to_id}")
            return True
            
        except errors.SerializationFailure:
            # Transaction conflict! Another transaction touched same data.
            # CockroachDB requires client-side retry.
            conn.rollback()
            if attempt < max_retries - 1:
                time.sleep(0.01 * (2 ** attempt))  # Exponential backoff
                continue
            raise
        except Exception:
            conn.rollback()
            raise
    
    return False

# Multi-region read: CockroachDB automatically routes to nearest replica
def get_account_balance(account_id):
    """Reads are served from the nearest replica (follower reads)"""
    cursor = conn.cursor()
    # AS OF SYSTEM TIME: read slightly stale data from local replica
    # (avoids cross-region latency for non-critical reads)
    cursor.execute(
        "SELECT balance FROM accounts AS OF SYSTEM TIME '-5s' WHERE id = %s",
        (account_id,)
    )
    return cursor.fetchone()[0]
```

### Java — TiDB (MySQL-compatible)

```java
import java.sql.*;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class TiDBExample {
    
    private final HikariDataSource dataSource;
    
    public TiDBExample() {
        // TiDB is MySQL-compatible — use MySQL JDBC driver!
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://tidb-server:4000/myapp?useSSL=true");
        config.setUsername("root");
        config.setPassword("secret");
        config.setMaximumPoolSize(20);
        
        // TiDB-specific: enable batch DML for bulk operations
        config.addDataSourceProperty("rewriteBatchedStatements", "true");
        
        this.dataSource = new HikariDataSource(config);
    }
    
    /**
     * Distributed transaction in TiDB — same SQL as MySQL!
     * But internally, TiDB coordinates across multiple TiKV nodes.
     */
    public void transferMoney(String fromAccount, String toAccount, 
                              double amount) throws SQLException {
        
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            // TiDB supports REPEATABLE READ and READ COMMITTED
            conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);
            
            try {
                // Pessimistic locking (TiDB 4.0+, same as MySQL FOR UPDATE)
                PreparedStatement lockStmt = conn.prepareStatement(
                    "SELECT balance FROM accounts WHERE id = ? FOR UPDATE");
                
                // Lock sender
                lockStmt.setString(1, fromAccount);
                ResultSet rs = lockStmt.executeQuery();
                rs.next();
                double senderBalance = rs.getDouble("balance");
                
                if (senderBalance < amount) {
                    conn.rollback();
                    throw new InsufficientFundsException(fromAccount, amount);
                }
                
                // Deduct from sender
                PreparedStatement deduct = conn.prepareStatement(
                    "UPDATE accounts SET balance = balance - ? WHERE id = ?");
                deduct.setDouble(1, amount);
                deduct.setString(2, fromAccount);
                deduct.executeUpdate();
                
                // Add to receiver
                PreparedStatement add = conn.prepareStatement(
                    "UPDATE accounts SET balance = balance + ? WHERE id = ?");
                add.setDouble(1, amount);
                add.setString(2, toAccount);
                add.executeUpdate();
                
                // Record transaction
                PreparedStatement log = conn.prepareStatement(
                    "INSERT INTO transactions (from_acc, to_acc, amount, ts) " +
                    "VALUES (?, ?, ?, NOW())");
                log.setString(1, fromAccount);
                log.setString(2, toAccount);
                log.setDouble(3, amount);
                log.executeUpdate();
                
                conn.commit();  // Distributed commit across TiKV nodes!
                
            } catch (SQLException e) {
                conn.rollback();
                throw e;
            }
        }
    }
    
    /**
     * HTAP: Same query, TiDB automatically uses TiFlash for analytics
     * when the optimizer determines columnar storage is faster.
     */
    public Map<String, Double> getDailyRevenue(String startDate, String endDate) 
            throws SQLException {
        
        Map<String, Double> revenue = new LinkedHashMap<>();
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 // This query automatically routes to TiFlash (columnar)
                 // if TiFlash replica exists — no code change needed!
                 "SELECT DATE(created_at) as day, SUM(amount) as total " +
                 "FROM orders " +
                 "WHERE created_at BETWEEN ? AND ? " +
                 "GROUP BY DATE(created_at) " +
                 "ORDER BY day")) {
            
            stmt.setString(1, startDate);
            stmt.setString(2, endDate);
            
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                revenue.put(rs.getString("day"), rs.getDouble("total"));
            }
        }
        return revenue;
    }
}
```

---

## Infrastructure Examples

### CockroachDB Multi-Region Deployment (Kubernetes)

```yaml
# CockroachDB Operator deployment — 9 nodes across 3 regions
apiVersion: crdb.cockroachlabs.com/v1alpha1
kind: CrdbCluster
metadata:
  name: cockroach-global
spec:
  dataStore:
    pvc:
      spec:
        storageClassName: premium-ssd
        resources:
          requests:
            storage: 500Gi
  nodes: 9  # 3 per region
  topology:
    - name: us-east-1
      nodeCount: 3
      locality: "region=us-east,zone=us-east-1"
    - name: eu-west-1
      nodeCount: 3
      locality: "region=eu-west,zone=eu-west-1"
    - name: ap-south-1
      nodeCount: 3
      locality: "region=ap-south,zone=ap-south-1"
  cockroachDBVersion: "v23.2.0"
  additionalArgs:
    - "--max-sql-memory=4GiB"
    - "--cache=4GiB"
---
# Zone configuration for geo-partitioning
# (applied via SQL after cluster is running)
# ALTER DATABASE myapp SET PRIMARY REGION "us-east";
# ALTER DATABASE myapp ADD REGION "eu-west";
# ALTER DATABASE myapp ADD REGION "ap-south";
# ALTER TABLE accounts SET LOCALITY REGIONAL BY ROW;
```

### TiDB Cluster (Docker Compose for Dev)

```yaml
version: '3.8'
services:
  pd0:
    image: pingcap/pd:v7.5.0
    command:
      - --name=pd0
      - --client-urls=http://0.0.0.0:2379
      - --peer-urls=http://0.0.0.0:2380
      - --initial-cluster=pd0=http://pd0:2380
    ports: ["2379:2379"]

  tikv0:
    image: pingcap/tikv:v7.5.0
    command:
      - --addr=0.0.0.0:20160
      - --advertise-addr=tikv0:20160
      - --pd=pd0:2379
    depends_on: [pd0]

  tikv1:
    image: pingcap/tikv:v7.5.0
    command:
      - --addr=0.0.0.0:20160
      - --advertise-addr=tikv1:20160
      - --pd=pd0:2379
    depends_on: [pd0]

  tikv2:
    image: pingcap/tikv:v7.5.0
    command:
      - --addr=0.0.0.0:20160
      - --advertise-addr=tikv2:20160
      - --pd=pd0:2379
    depends_on: [pd0]

  tidb:
    image: pingcap/tidb:v7.5.0
    command:
      - --store=tikv
      - --path=pd0:2379
    ports: ["4000:4000"]  # MySQL port!
    depends_on: [tikv0, tikv1, tikv2]

  # TiFlash for HTAP (analytics on same data)
  tiflash:
    image: pingcap/tiflash:v7.5.0
    command:
      - --path=/data
      - --pd=pd0:2379
    depends_on: [tikv0, tikv1, tikv2]
```

### Google Cloud Spanner (Terraform)

```hcl
resource "google_spanner_instance" "main" {
  name         = "global-banking"
  config       = "nam-eur-asia1"  # Multi-continent!
  display_name = "Global Banking Instance"
  num_nodes    = 3  # Minimum for multi-region
  
  labels = {
    env = "production"
  }
}

resource "google_spanner_database" "banking" {
  instance = google_spanner_instance.main.name
  name     = "banking"
  
  ddl = [
    <<-SQL
      CREATE TABLE Accounts (
        AccountId   STRING(36) NOT NULL,
        OwnerId     STRING(36) NOT NULL,
        Balance     NUMERIC NOT NULL,
        Currency    STRING(3) NOT NULL,
        Region      STRING(20) NOT NULL,
        CreatedAt   TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
      ) PRIMARY KEY (AccountId)
    SQL,
    <<-SQL
      CREATE TABLE Transactions (
        TransactionId STRING(36) NOT NULL,
        FromAccount   STRING(36) NOT NULL,
        ToAccount     STRING(36) NOT NULL,
        Amount        NUMERIC NOT NULL,
        Timestamp     TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
      ) PRIMARY KEY (TransactionId),
      INTERLEAVE IN PARENT Accounts ON DELETE CASCADE
    SQL,
  ]
  
  deletion_protection = true
}
```

---

## Real-World Example

### Google — Spanner Powering Global Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│           Google Spanner — Real Production Usage                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Google Ads (primary use case):                                  │
│  • Stores advertiser budgets and billing                        │
│  • 10+ million QPS globally                                     │
│  • ACID transactions: "never overspend a budget"               │
│  • Multi-region: ad serving continues if entire region goes down│
│                                                                   │
│  Google Play:                                                    │
│  • App purchases, subscriptions, refunds                        │
│  • Must be ACID (real money!)                                   │
│  • Must be global (users everywhere)                            │
│                                                                   │
│  Architecture:                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              5 Continents, 7+ Regions                      │  │
│  │                                                            │  │
│  │  US-East ←──Paxos──→ US-West ←──Paxos──→ Europe          │  │
│  │    ↕                    ↕                    ↕             │  │
│  │  Asia  ←──Paxos──→  Oceania                              │  │
│  │                                                            │  │
│  │  Write: Goes to Paxos leader → replicated to majority    │  │
│  │  Read: Served from ANY replica (with TrueTime timestamp) │  │
│  │                                                            │  │
│  │  Performance:                                              │  │
│  │  • Single-region write: ~6ms                              │  │
│  │  • Cross-region write: ~20-50ms (Paxos round-trip)       │  │
│  │  • Strong read: ~1ms (from local replica!)               │  │
│  │  • Stale read (10s): <1ms (no coordination needed)       │  │
│  │                                                            │  │
│  │  Scale:                                                    │  │
│  │  • Petabytes of data                                      │  │
│  │  • Billions of rows per table                             │  │
│  │  • 99.999% availability (5 nines = <5min downtime/year)  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  What makes it possible:                                         │
│  • TrueTime: hardware (GPS + atomic clocks) in every DC        │
│  • Commit-wait: wait out uncertainty before confirming          │
│  • Paxos: consensus without single point of failure             │
│  • Automatic resharding: splits hot ranges without downtime     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using NewSQL for a small single-region app | Massive overhead for no benefit | Use PostgreSQL until you outgrow one server |
| Sequential INT primary keys in CockroachDB | Creates "hotspot" — all writes go to one range! | Use UUID or HASH-sharded indexes |
| Ignoring cross-region latency | Writes require consensus (50-100ms cross-region) | Place leader replicas near write-heavy regions |
| Treating it as "just PostgreSQL/MySQL" | Distributed transactions have different retry semantics | Handle serialization conflicts with retry loops |
| Not using follower reads for reads | Every read goes to leader (unnecessary latency) | Use AS OF SYSTEM TIME for slightly-stale but fast reads |
| Undersizing cluster | 3 nodes minimum for Raft consensus, need headroom | Run 5+ nodes in production for fault tolerance |
| Wide transactions (touching many ranges) | More ranges = more coordination = slower commits | Minimize transaction scope, batch by range |

---

## When to Use / When NOT to Use

### ✅ Use NewSQL When:
- Need **ACID + horizontal scale** (can't choose between SQL and NoSQL)
- **Global application** with users on multiple continents requiring consistent data
- **Financial/banking** data that must be correct AND highly available
- Outgrowing PostgreSQL's single-node limits (TB+ data, 100K+ writes/sec)
- Need **automatic failover** without data loss (RPO=0)
- Want **SQL compatibility** while scaling horizontally

### ❌ Do NOT Use NewSQL When:
- **Single region, single server is enough** — PostgreSQL/MySQL is simpler and faster
- **Read-heavy, write-light** — read replicas on PostgreSQL are simpler
- **Eventual consistency is acceptable** — Cassandra/DynamoDB are simpler for that
- **Analytics/OLAP only** — ClickHouse, BigQuery are better for pure analytics
- **Budget is tight** — NewSQL clusters need minimum 3-5 nodes (costly)
- **Team lacks distributed systems expertise** — operational complexity is real
- **< 1TB data, < 10K writes/sec** — PostgreSQL handles this easily

### Decision Framework:

```
Do you need ACID transactions?
├── NO → Use DynamoDB/Cassandra (NoSQL)
└── YES → Can PostgreSQL handle your scale?
    ├── YES → Use PostgreSQL (simplest solution!)
    └── NO → Do you need multi-region?
        ├── NO → Consider Citus (distributed PostgreSQL)
        └── YES → Do you need MySQL compatibility?
            ├── YES → TiDB
            └── NO → CockroachDB (PostgreSQL-compatible)
                     or Cloud Spanner (Google ecosystem)
```

---

## Key Takeaways

1. **NewSQL = SQL + NoSQL combined** — distributed horizontal scaling with full ACID transactions. The trade-off is latency (consensus rounds) and operational complexity.
2. **Google Spanner** pioneered NewSQL using TrueTime (GPS + atomic clocks) to achieve external consistency globally. Available as Cloud Spanner (managed).
3. **CockroachDB** achieves similar guarantees without special hardware, using Hybrid Logical Clocks and Raft consensus. PostgreSQL-compatible SQL.
4. **TiDB** is MySQL-compatible NewSQL with a unique HTAP capability (TiFlash) — run analytics on live transactional data without ETL.
5. **Don't use NewSQL prematurely** — PostgreSQL handles most applications. NewSQL shines when you need ACID at global scale (multi-region, TB+ data, 100K+ writes/sec).
6. **Handle retries at the application level** — distributed transactions may conflict. Always implement retry logic with exponential backoff.
7. **Use follower reads for latency-sensitive reads** — not every read needs the absolute latest data. Reading "5 seconds stale" from a local replica avoids cross-region round trips.

---

## What's Next?

Congratulations! You've completed **Part 9: Databases — Storing, Querying & Managing Data**. You now understand everything from basic relational databases to planet-scale NewSQL systems.

Next up is **Part 10: Caching Strategies**, where you'll learn how to dramatically reduce database load and serve responses in microseconds using caching at every layer of your architecture.
