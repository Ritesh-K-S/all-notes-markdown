# 🔗 Chapter 5.4 — Connection Pooling & Database Proxies

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** DBMS Architecture (Chapter 1.3), any backend development experience

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why connection pooling is essential** — not optional
- Know the **cost of a database connection** (memory, CPU, time)
- Master the **top connection poolers**: HikariCP, PgBouncer, ProxySQL, c3p0
- Calculate the **optimal pool size** using a battle-tested formula
- Understand **database proxies** and when to use them
- Diagnose **connection leaks, pool exhaustion, and timeout issues** like a pro

---

## 🧠 The Problem — Why Connection Pooling Matters

### What Happens When You Open a Database Connection?

```
  App sends "I want to connect" to database:

  ┌──────────┐                                    ┌──────────┐
  │   App    │ ──── TCP SYN ────────────────────► │ Database │
  │          │ ◄─── TCP SYN-ACK ─────────────────  │          │
  │          │ ──── TCP ACK ────────────────────►  │          │
  │          │                                     │          │
  │          │ ──── SSL Handshake (if TLS) ──────► │          │
  │          │ ◄─── Certificate + Key Exchange ──  │          │
  │          │                                     │          │
  │          │ ──── Authentication ──────────────► │          │
  │          │ ◄─── Auth OK ─────────────────────  │          │
  │          │                                     │          │
  │          │ ──── Session Setup ───────────────► │          │
  │          │ ◄─── Ready for Query ─────────────  │          │
  └──────────┘                                    └──────────┘

  Time: 20-100ms per connection (even on local network!)

  Resources allocated PER connection on the database:
  ┌────────────────────────────────────────────────────────────┐
  │  PostgreSQL:  ~10 MB RAM per connection (process-based)    │
  │  MySQL:       ~1-4 MB RAM per connection (thread-based)    │
  │  Oracle:      ~5-10 MB per connection (dedicated server)   │
  │  SQL Server:  ~1-3 MB per connection (thread-based)        │
  │  MongoDB:     ~1 MB per connection                         │
  └────────────────────────────────────────────────────────────┘
```

### The Disaster Without Pooling

```
  Web Server handling 500 concurrent requests:

  ❌ WITHOUT Connection Pooling:

  Request 1  → Open Connection → Query → Close Connection    (20ms overhead)
  Request 2  → Open Connection → Query → Close Connection    (20ms overhead)
  Request 3  → Open Connection → Query → Close Connection    (20ms overhead)
  ...
  Request 500 → Open Connection → Query → Close Connection   (20ms overhead)

  Problems:
  ├── 500 connections × 10MB = 5GB RAM consumed on DB server! 💀
  ├── 500 × 20ms = 10 seconds of pure overhead (no actual work!)
  ├── TCP port exhaustion (each connection = 1 socket)
  ├── Database says "max_connections reached" → new requests FAIL
  └── Database CPU spends time forking processes, not running queries

  ✅ WITH Connection Pooling:

  ┌──────────────────┐     ┌───────────────┐     ┌──────────────┐
  │   App Server     │     │  Connection   │     │   Database   │
  │   (500 requests) │────►│    Pool       │────►│  (20 conns)  │
  │                  │     │  (20 conns)   │     │              │
  │  Request 1 ──►  │     │  ┌──┐ ┌──┐    │     │  20 × 10MB   │
  │  Request 2 ──►  │     │  │C1│ │C2│    │     │  = 200MB ✅   │
  │  Request 3 ──►  │     │  └──┘ └──┘    │     │              │
  │  ...            │     │  ┌──┐ ┌──┐    │     │              │
  │  Request 500 ►  │     │  │C3│ │C4│    │     │              │
  │                  │     │  └──┘ └──┘    │     │              │
  │  500 requests   │     │  ...          │     │              │
  │  share 20 conns │     │  ┌──┐         │     │              │
  │                  │     │  │C20│        │     │              │
  │                  │     │  └──┘         │     │              │
  └──────────────────┘     └───────────────┘     └──────────────┘

  Results:
  ├── 20 connections × 10MB = 200MB RAM (not 5GB!) ✅
  ├── No connection creation overhead (reuse existing) ✅
  ├── Requests queue when all connections busy ✅
  ├── Database stays healthy and responsive ✅
  └── 25x less memory, 50x less connection overhead ✅
```

> **Connection pooling is NOT an optimization. It's a REQUIREMENT for any production application.**

---

## 📦 How Connection Pooling Works

### The Pool Lifecycle

```
                    Connection Pool Lifecycle

  ┌─────────────────────────────────────────────────────────────┐
  │                    CONNECTION POOL                          │
  │                                                             │
  │  Pool Config:                                               │
  │  ├── Min Size: 5 (always keep 5 connections warm)           │
  │  ├── Max Size: 20 (never exceed 20 connections)             │
  │  ├── Idle Timeout: 10 min (close idle connections after)    │
  │  └── Max Lifetime: 30 min (recycle connections)             │
  │                                                             │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
  │  │  IDLE    │  │  IDLE    │  │ IN USE   │  │ IN USE   │   │
  │  │  conn 1  │  │  conn 2  │  │  conn 3  │  │  conn 4  │   │
  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
  │  ┌──────────┐                                               │
  │  │  IDLE    │  ← 3 idle, 2 in use = healthy pool           │
  │  │  conn 5  │                                               │
  │  └──────────┘                                               │
  └─────────────────────────────────────────────────────────────┘

  Request Flow:
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  App Request                                                │
  │       │                                                     │
  │       ▼                                                     │
  │  [Borrow Connection from Pool]                              │
  │       │                                                     │
  │       ├── Idle connection available? ──YES──► Return it     │
  │       │                                      (< 1ms!)      │
  │       │                                                     │
  │       ├── Pool at max size? ──YES──► WAIT in queue          │
  │       │                              (up to timeout)        │
  │       │                                                     │
  │       └── Pool NOT full? ──YES──► Create new connection     │
  │                                   Add to pool, return it    │
  │                                                             │
  │  [Use Connection]                                           │
  │       │                                                     │
  │       ▼                                                     │
  │  Execute queries...                                         │
  │       │                                                     │
  │       ▼                                                     │
  │  [Return Connection to Pool]                                │
  │       │                                                     │
  │       ├── Connection healthy? ──YES──► Mark as IDLE         │
  │       │                                                     │
  │       └── Connection broken? ──YES──► Destroy, create new   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Connection Pooler Deep Dives

### 1. HikariCP — The Fastest JVM Connection Pool ⭐🔥

> **Used by:** Spring Boot (default since 2.0), every serious Java application
> **Performance:** 100x faster than c3p0, 50x faster than Tomcat DBCP
> **Name:** 光 (Hikari) = "Light" in Japanese

```yaml
# Spring Boot — application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: app_user
    password: ${DB_PASSWORD}
    hikari:
      pool-name: MyApp-Pool
      minimum-idle: 5              # Min connections to keep warm
      maximum-pool-size: 20        # Max connections ever
      idle-timeout: 600000         # 10 min — close idle connections
      max-lifetime: 1800000        # 30 min — recycle all connections
      connection-timeout: 30000    # 30s — max wait for a connection
      leak-detection-threshold: 60000  # Warn if connection held > 60s
      validation-timeout: 5000    # 5s — how long to validate a conn
      connection-test-query: SELECT 1  # For non-JDBC4 drivers
```

```java
// Manual HikariCP Configuration (without Spring Boot)
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/myapp");
config.setUsername("app_user");
config.setPassword(System.getenv("DB_PASSWORD"));
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setIdleTimeout(600_000);
config.setMaxLifetime(1_800_000);
config.setConnectionTimeout(30_000);
config.setLeakDetectionThreshold(60_000);

// Performance optimizations
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
config.addDataSourceProperty("useServerPrepStmts", "true");

HikariDataSource ds = new HikariDataSource(config);

// Use it
try (Connection conn = ds.getConnection()) {   // Borrow from pool
    PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    stmt.setInt(1, 42);
    ResultSet rs = stmt.executeQuery();
    // Process results...
}  // Connection auto-returned to pool (try-with-resources)
```

#### HikariCP Key Metrics to Monitor

```
  HikariCP exposes JMX/Micrometer metrics:

  ┌────────────────────────────────┬────────────────────────────────┐
  │ Metric                        │ What It Means                  │
  ├────────────────────────────────┼────────────────────────────────┤
  │ hikaricp_connections_active    │ Currently in use               │
  │ hikaricp_connections_idle      │ Waiting to be used             │
  │ hikaricp_connections_total     │ Total in pool                  │
  │ hikaricp_connections_pending   │ Threads waiting for connection │
  │ hikaricp_connections_timeout   │ Timed out waiting ⚠️           │
  │ hikaricp_connections_creation  │ Time to create new connection  │
  │ hikaricp_connections_usage     │ Time connection was borrowed   │
  └────────────────────────────────┴────────────────────────────────┘

  🚨 If hikaricp_connections_pending > 0 for sustained periods:
     → Pool is too small OR queries are too slow!
```

---

### 2. PgBouncer — PostgreSQL's External Connection Pooler ⭐🔥

> **Why PgBouncer?** PostgreSQL creates a new **OS process** per connection (~10MB each).
> 1000 connections = 10GB RAM just for process overhead!
> PgBouncer multiplexes 1000 app connections into ~50 actual PG connections.

```
  Without PgBouncer:

  App Server 1 (100 conns) ──┐
  App Server 2 (100 conns) ──┼──► PostgreSQL (300 connections = 3GB RAM!)
  App Server 3 (100 conns) ──┘

  With PgBouncer:

  App Server 1 (100 conns) ──┐
  App Server 2 (100 conns) ──┼──► PgBouncer (300→50) ──► PostgreSQL (50 conns = 500MB!)
  App Server 3 (100 conns) ──┘
                                    Multiplexes!
```

#### PgBouncer Pool Modes

```
  ┌──────────────────────────────────────────────────────────────┐
  │  MODE              WHAT IT DOES              BEST FOR        │
  ├──────────────────────────────────────────────────────────────┤
  │                                                              │
  │  SESSION           Connection assigned for    Apps using      │
  │  (safest)          entire client session.     prepared stmts, │
  │                    Released on disconnect.    temp tables,     │
  │                    Least multiplexing.        session vars     │
  │                                                              │
  │  TRANSACTION       Connection assigned for    Most web apps!  │
  │  (recommended) ⭐  single transaction.       Stateless APIs,  │
  │                    Released after COMMIT.     microservices    │
  │                    Great multiplexing.                        │
  │                                                              │
  │  STATEMENT         Connection assigned for    Very simple     │
  │  (aggressive)      single statement.          apps with no    │
  │                    Best multiplexing.         transactions    │
  │                    ⚠️ No multi-statement txns!                 │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

#### PgBouncer Configuration

```ini
; /etc/pgbouncer/pgbouncer.ini

[databases]
myapp = host=db-primary.internal port=5432 dbname=myapp

[pgbouncer]
; Listening config
listen_addr = 0.0.0.0
listen_port = 6432                    ; PgBouncer port (NOT 5432!)

; Pool mode
pool_mode = transaction               ; ⭐ Recommended for most apps

; Pool sizing
default_pool_size = 50                ; Connections per user/database pair
min_pool_size = 10                    ; Minimum idle connections to keep
max_client_conn = 1000                ; Max client connections to PgBouncer
max_db_connections = 100              ; Max total connections to PostgreSQL
reserve_pool_size = 5                 ; Emergency reserve connections
reserve_pool_timeout = 3              ; Seconds before using reserve

; Timeouts
server_idle_timeout = 600             ; Close idle server connections (10 min)
client_idle_timeout = 0               ; Don't close idle clients (0 = disabled)
query_timeout = 0                     ; No query timeout (handled by app)
query_wait_timeout = 120              ; Max wait for connection (2 min)

; Connection health
server_reset_query = DISCARD ALL      ; Reset connection state on return
server_check_query = SELECT 1         ; Health check query
server_check_delay = 30               ; Seconds between health checks

; Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60                     ; Log stats every 60 seconds

; Authentication
auth_type = md5                       ; or scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
```

```bash
# PgBouncer admin commands (connect to PgBouncer admin console)
psql -h localhost -p 6432 -U pgbouncer pgbouncer

SHOW POOLS;        # Current pool status
SHOW STATS;        # Connection statistics
SHOW SERVERS;      # Backend PostgreSQL connections
SHOW CLIENTS;      # Client connections
SHOW CONFIG;       # Current configuration
RELOAD;            # Reload config without restart
PAUSE myapp;       # Pause all queries (for maintenance)
RESUME myapp;      # Resume queries
```

```
  PgBouncer SHOW POOLS output:

  database │ user     │ cl_active │ cl_waiting │ sv_active │ sv_idle │ pool_mode
  ─────────┼──────────┼───────────┼────────────┼───────────┼─────────┼──────────
  myapp    │ app_user │    45     │     0      │    12     │   38    │ transaction

  cl_active:  45 app connections currently in a transaction
  cl_waiting:  0 app connections waiting (good! if >0, pool may be too small)
  sv_active:  12 actual PostgreSQL connections doing work
  sv_idle:    38 actual PostgreSQL connections idle (ready for reuse)

  45 app connections → only 12 actual DB connections! That's 73% multiplexing! ✅
```

---

### 3. ProxySQL — MySQL's Swiss Army Proxy ⭐

> **What:** High-performance MySQL proxy + connection pooler + query router
> **Used by:** Large MySQL deployments at scale

```
  ProxySQL Architecture:

  ┌──────────────┐     ┌───────────────────────────┐     ┌──────────────┐
  │  App Servers │     │       ProxySQL              │     │  MySQL       │
  │              │     │                             │     │  Servers     │
  │  Server 1 ──┼────►│  ┌──────────────────────┐  │     │              │
  │  Server 2 ──┼────►│  │ Connection Pooling   │  │────►│  Primary     │
  │  Server 3 ──┼────►│  │ Query Routing        │  │     │  (writes)    │
  │              │     │  │ Read/Write Split     │──┼────►│              │
  │              │     │  │ Query Caching        │  │     │  Replica 1   │
  │              │     │  │ Query Firewall       │──┼────►│  (reads)     │
  │              │     │  └──────────────────────┘  │     │              │
  │              │     │                             │     │  Replica 2   │
  └──────────────┘     └───────────────────────────┘     │  (reads)     │
                                                          └──────────────┘
```

```sql
-- ProxySQL Read/Write Splitting Configuration

-- Define backend servers
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES
    (10, 'mysql-primary.internal', 3306, 1),    -- Writer (hostgroup 10)
    (20, 'mysql-replica1.internal', 3306, 1),   -- Reader (hostgroup 20)
    (20, 'mysql-replica2.internal', 3306, 1);   -- Reader (hostgroup 20)

-- Route queries automatically
INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup) VALUES
    (1, '^SELECT.*FOR UPDATE', 10),              -- SELECT FOR UPDATE → primary
    (2, '^SELECT', 20),                          -- All other SELECTs → replica
    (3, '.*', 10);                               -- Everything else → primary

LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;
```

---

### 4. Application-Level Poolers Comparison

| Pooler | Language/Platform | Key Feature | Default Pool Size |
|--------|-------------------|-------------|-------------------|
| **HikariCP** | Java/JVM | Fastest JVM pool, Spring Boot default | 10 |
| **c3p0** | Java/JVM | Older, still used in legacy apps | 3 |
| **Tomcat DBCP** | Java/JVM | Part of Tomcat, good for web apps | 8 |
| **pgx Pool** | Go | Built into pgx PostgreSQL driver | 4 |
| **asyncpg** | Python | Async PostgreSQL, very fast | 10 |
| **SQLAlchemy Pool** | Python | Built into SQLAlchemy engine | 5 |
| **node-postgres Pool** | Node.js | Built into pg module | 10 |
| **Npgsql Pool** | .NET | Built into Npgsql driver | 100 (min 0) |
| **Prisma Pool** | Node.js | Managed by Prisma engine | `num_cpus * 2 + 1` |

---

## 🧮 Pool Size Formula — The Most Important Number

### The Golden Formula (from HikariCP creator)

```
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │     Pool Size = (Number of CPU Cores × 2) + Effective Spindle Count      │
  │                                                                │
  │     For SSD (no spindles):                                     │
  │     Pool Size = (CPU Cores × 2) + 1                           │
  │                                                                │
  │     Example: 4-core server with SSD                           │
  │     Pool Size = (4 × 2) + 1 = 9 connections                  │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

  Wait... only 9?! For 10,000 concurrent users?!

  YES! And here's why:
```

### Why Smaller Pools Are FASTER

```
  ❌ Common Misconception:
  "More connections = more throughput"

  ✅ Reality:
  "More connections = more contention = SLOWER"

  Proof:

  Scenario: 4-core CPU, 10,000 requests/second

  Pool Size = 200:
  ┌──────────────────────────────────────────────────────────────┐
  │  200 connections all competing for 4 CPU cores               │
  │  200 connections all competing for disk I/O                  │
  │  Massive context switching overhead                          │
  │  Lock contention on shared resources                         │
  │  Result: 8,000 requests/second (CPU busy context switching)  │
  └──────────────────────────────────────────────────────────────┘

  Pool Size = 10:
  ┌──────────────────────────────────────────────────────────────┐
  │  10 connections, 4 active at a time                          │
  │  Minimal context switching                                   │
  │  No lock contention                                          │
  │  Queries complete faster → connections return faster          │
  │  Result: 12,000 requests/second (CPU does useful work!) ⚡   │
  └──────────────────────────────────────────────────────────────┘

  PostgreSQL Benchmark (real numbers):
  ┌────────────────┬──────────────────┐
  │ Pool Size      │ Transactions/sec │
  ├────────────────┼──────────────────┤
  │ 600 connections│     1,200 TPS    │
  │ 100 connections│     5,800 TPS    │
  │ 50 connections │     9,200 TPS    │
  │ 10 connections │    11,500 TPS ⭐ │
  │ 5 connections  │    10,800 TPS    │
  └────────────────┴──────────────────┘
  Sweet spot: ~10 connections for a 4-core machine!
```

### Multi-Server Pool Sizing

```
  Total DB connections available: 100 (max_connections)
  Reserve for admin/monitoring: 10
  Available for app: 90

  Number of app servers: 5

  Pool size per server = 90 / 5 = 18 connections each

  ┌──────────┐
  │ App Srv 1│── 18 conns ──┐
  │ App Srv 2│── 18 conns ──┤
  │ App Srv 3│── 18 conns ──┼──► PostgreSQL (90 app + 10 admin = 100)
  │ App Srv 4│── 18 conns ──┤
  │ App Srv 5│── 18 conns ──┘
  └──────────┘

  ⚠️ Don't forget:
  ├── Each microservice needs its OWN pool
  ├── Background workers need connections too
  ├── Admin tools (pgAdmin, monitoring) need connections
  └── Leave headroom for spikes!
```

---

## 🏗️ Architecture Patterns

### Pattern 1: Application-Level Pooling (Simple)

```
  ┌──────────────┐     ┌──────────┐
  │ App Server   │     │ Database │
  │              │     │          │
  │  ┌────────┐  │     │          │
  │  │HikariCP│──┼────►│          │
  │  │Pool    │  │     │          │
  │  │(20)    │  │     │          │
  │  └────────┘  │     │          │
  └──────────────┘     └──────────┘

  ✅ Simplest setup
  ✅ No additional infrastructure
  ❌ Each app instance has its own pool
  ❌ Hard to control total connection count across all instances
```

### Pattern 2: External Pooler (Production)

```
  ┌──────────┐     ┌──────────────┐     ┌──────────┐
  │ App Srv 1│────►│              │     │          │
  │ (pool:20)│     │  PgBouncer   │────►│PostgreSQL│
  │ App Srv 2│────►│              │     │ (50 real │
  │ (pool:20)│     │  (1000 → 50) │     │  conns)  │
  │ App Srv 3│────►│              │     │          │
  │ (pool:20)│     │              │     │          │
  └──────────┘     └──────────────┘     └──────────┘

  ✅ Centralized connection management
  ✅ Massive multiplexing (1000 → 50)
  ✅ Can pause connections for maintenance
  ❌ Additional infrastructure to manage
  ❌ Slight latency added (usually <1ms)
```

### Pattern 3: Sidecar Proxy (Kubernetes/Cloud Native)

```
  ┌─────────────────────────── Pod ──────────────────────────────┐
  │                                                              │
  │  ┌──────────────┐     ┌─────────────────────┐              │
  │  │ App Container│────►│ Cloud SQL Auth Proxy │──► Cloud DB  │
  │  │              │     │ (or PgBouncer sidecar)│              │
  │  └──────────────┘     └─────────────────────┘              │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘

  Used in: GCP Cloud SQL, AWS RDS Proxy, Azure connection pooling
  ✅ Per-pod isolation
  ✅ Handles auth automatically
  ✅ Cloud-native SSL/IAM integration
```

### Cloud-Managed Connection Poolers

| Service | Provider | Database | Key Feature |
|---------|----------|----------|-------------|
| **RDS Proxy** | AWS | MySQL, PostgreSQL | Managed PgBouncer, IAM auth |
| **Cloud SQL Auth Proxy** | GCP | MySQL, PostgreSQL, SQL Server | IAM auth, SSL automatic |
| **Azure SQL Connection Pooling** | Azure | SQL Server, PostgreSQL | Built-in, no config needed |
| **PlanetScale Connection Pooling** | PlanetScale | MySQL (Vitess) | Edge-aware, global |
| **Supabase Pooler** | Supabase | PostgreSQL | PgBouncer managed, transaction mode |
| **Neon Connection Pooler** | Neon | PostgreSQL | Serverless-optimized |

---

## 🐛 Troubleshooting Connection Issues

### Problem 1: Connection Pool Exhaustion

```
  Symptoms:
  └── "Unable to acquire connection from pool"
  └── "Connection pool timeout after 30000ms"
  └── Requests hanging, then timing out
  └── Application freezes under load

  Cause: All connections are in use, new requests can't get one

  ┌────────────────────────────────────────────────────────────────┐
  │ Pool: [IN USE] [IN USE] [IN USE] [IN USE] [IN USE]           │
  │       [IN USE] [IN USE] [IN USE] [IN USE] [IN USE]           │
  │                                                                │
  │ Queue: [waiting...] [waiting...] [waiting...] [TIMEOUT! ☠️]  │
  └────────────────────────────────────────────────────────────────┘

  Fixes:
  1. Find slow queries (they hold connections longer)
     → Check slow query log, enable pg_stat_statements
  2. Find connection leaks (code that borrows but never returns)
     → Enable leak detection (HikariCP: leakDetectionThreshold)
  3. Reduce query execution time (add indexes, optimize queries)
  4. LAST RESORT: increase pool size (only if CPU/memory allows)
```

### Problem 2: Connection Leak

```java
// ❌ CONNECTION LEAK — Connection never returned to pool!
public User getUser(int id) {
    Connection conn = dataSource.getConnection();  // Borrowed from pool
    PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    stmt.setInt(1, id);
    ResultSet rs = stmt.executeQuery();
    User user = mapUser(rs);
    return user;
    // 💀 conn.close() never called! Connection stuck forever!
    // Pool slowly empties. App eventually dies.
}

// ✅ FIXED — Use try-with-resources (Java 7+)
public User getUser(int id) {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
        stmt.setInt(1, id);
        try (ResultSet rs = stmt.executeQuery()) {
            return mapUser(rs);
        }
    }  // Connection auto-returned to pool, even if exception occurs! ✅
}
```

```python
# ❌ CONNECTION LEAK — Python
conn = engine.connect()
result = conn.execute(text("SELECT * FROM users"))
# Forgot conn.close()! 💀

# ✅ FIXED — Context manager
with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users"))
# Auto-closed! ✅
```

### Problem 3: Stale Connections

```
  Symptom: "Connection is closed" errors randomly

  Cause: Network timeout, firewall killed idle connections,
         database restarted, but pool still holds dead connections

  ┌──────────┐     ┌──────────────┐     ┌──────────┐
  │   Pool   │     │  Firewall    │     │ Database │
  │          │     │              │     │          │
  │ conn 1 ──┼──X──┼── killed ───┼──   │          │
  │ conn 2 ──┼─────┼── alive ────┼────►│          │
  │ conn 3 ──┼──X──┼── killed ───┼──   │          │
  └──────────┘     └──────────────┘     └──────────┘

  Fixes:
  ├── Set maxLifetime < firewall/DB timeout
  │   (HikariCP default: 30 min, adjust to match infra)
  ├── Enable connection validation on borrow
  │   (connectionTestQuery = "SELECT 1")
  ├── Set idleTimeout to recycle unused connections
  └── Use TCP keepalive (keeps firewalls happy)
```

### Diagnostic Queries

```sql
-- PostgreSQL: See all current connections
SELECT pid, usename, application_name, client_addr,
       state, query, query_start,
       now() - query_start AS query_duration
FROM pg_stat_activity
WHERE datname = 'myapp'
ORDER BY query_start;

-- PostgreSQL: Count connections by state
SELECT state, COUNT(*)
FROM pg_stat_activity
WHERE datname = 'myapp'
GROUP BY state;

-- PostgreSQL: Find long-running queries (holding connections!)
SELECT pid, usename, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '30 seconds'
ORDER BY duration DESC;

-- PostgreSQL: Kill a stuck connection
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid = 12345;

-- MySQL: See connections
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads%';
SHOW VARIABLES LIKE 'max_connections';

-- MySQL: Kill a connection
KILL 12345;
```

---

## 📊 Configuration Cheat Sheet

```
  Quick Reference — Recommended Settings:

  ┌─────────────────────────┬──────────────────────────────────────┐
  │ Parameter               │ Recommendation                       │
  ├─────────────────────────┼──────────────────────────────────────┤
  │ Max Pool Size           │ (CPU cores × 2) + 1                 │
  │ Min Idle                │ Same as max pool (avoid cold starts) │
  │ Connection Timeout      │ 30 seconds (fail fast)               │
  │ Idle Timeout            │ 10 minutes                           │
  │ Max Lifetime            │ 30 minutes (< DB wait_timeout)       │
  │ Leak Detection          │ 60 seconds (in dev/staging)          │
  │ Validation Query        │ SELECT 1 (or JDBC4 isValid())        │
  │ PgBouncer Pool Mode     │ transaction (for web apps)           │
  │ PgBouncer Max Client    │ 1000+ (PgBouncer handles thousands) │
  │ PgBouncer Default Pool  │ 50 (actual PG connections)           │
  └─────────────────────────┴──────────────────────────────────────┘
```

---

## 🏆 Key Takeaways

| Concept | Remember This |
|---------|---------------|
| **Connection cost** | 20-100ms to create, 1-10MB RAM each |
| **Pool = reuse** | Borrow, use, return — never create/destroy per request |
| **Pool size formula** | `(CPU cores × 2) + 1` — smaller is often FASTER |
| **HikariCP** | Fastest JVM pooler, Spring Boot default |
| **PgBouncer** | External pooler for PostgreSQL, transaction mode ⭐ |
| **ProxySQL** | MySQL proxy + pooler + query router |
| **Connection leaks** | Always use try-with-resources / context managers |
| **Pool exhaustion** | Fix slow queries first, increase size last |
| **Max lifetime** | Recycle connections before firewalls/DBs kill them |
| **External pooler** | Use when you have many app instances → 1 database |

---

> 🚀 **Next Up:** [Chapter 5.5 — Database Monitoring & Observability](./05-Monitoring.md) — Learn how to monitor your databases with Prometheus, Grafana, and never be surprised by a production incident again.

---

*Last Updated: June 2026*
