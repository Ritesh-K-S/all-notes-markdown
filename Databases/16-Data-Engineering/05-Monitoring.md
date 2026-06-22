# 📡 Chapter 5.5 — Database Monitoring & Observability

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~4-5 hours
> **Prerequisites:** DBMS Architecture (Chapter 1.3), basic Linux/command-line skills

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the **difference between monitoring and observability**
- Know the **critical metrics** every database needs monitored — no guessing
- Set up **Prometheus + Grafana** for database monitoring
- Master **built-in monitoring tools** for PostgreSQL, MySQL, Oracle, SQL Server, and MongoDB
- Build **alerting rules** that catch problems BEFORE users notice
- Perform **slow query analysis** across every major database
- Think like an **SRE/DBA** — proactive, not reactive

---

## 🧠 The Problem — Why Monitoring Is Non-Negotiable

### The 3 AM Horror Story

```
  Without Monitoring:

  2:00 AM   Database disk is 95% full (nobody knows)
  2:30 AM   Disk hits 100%
  2:31 AM   Database goes READ-ONLY (can't write)
  2:32 AM   All INSERT/UPDATE queries fail
  2:33 AM   App starts returning 500 errors
  2:45 AM   Customers tweet about outage
  3:00 AM   PagerDuty finally fires (customer complaints threshold)
  3:15 AM   Sleepy DBA logs in, finds disk full
  3:30 AM   Emergency cleanup
  4:00 AM   Service restored
  ─── Total downtime: 1.5 hours, reputation damaged ───

  With Monitoring:

  2:00 AM   Alert fires: "Disk usage > 85% on db-primary"
  2:05 AM   Auto-cleanup job triggers (purge old logs)
  2:10 AM   Disk usage drops to 72%
  2:15 AM   Alert resolves automatically
  ─── Total downtime: 0 minutes, nobody woke up ───
```

> **Monitoring doesn't prevent problems. It prevents SURPRISES.**

---

## 📊 Monitoring vs Observability — Know the Difference

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  MONITORING                        OBSERVABILITY                 │
  │  "What happened?"                  "Why did it happen?"          │
  │                                                                  │
  │  ┌─────────────────┐              ┌─────────────────────────┐   │
  │  │ Dashboard says   │              │ I can ask ANY question   │   │
  │  │ CPU is 95%      │              │ and find the answer      │   │
  │  │ Disk is 85%     │              │                         │   │
  │  │ Queries are slow│              │ "Why did query X slow   │   │
  │  │                 │              │  down at 2 PM?"          │   │
  │  │ Known-unknowns  │              │ "Which table is causing  │   │
  │  │ (we know what   │              │  most I/O?"              │   │
  │  │  to look at)    │              │                         │   │
  │  └─────────────────┘              │ Unknown-unknowns        │   │
  │                                    │ (we can explore freely) │   │
  │  Tools: Dashboards, Alerts        └─────────────────────────┘   │
  │                                                                  │
  │  Three Pillars of Observability:                                │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
  │  │ METRICS  │  │  LOGS    │  │  TRACES  │                     │
  │  │          │  │          │  │          │                     │
  │  │ Numbers  │  │ Events   │  │ Request  │                     │
  │  │ over time│  │ with     │  │ journey  │                     │
  │  │          │  │ context  │  │ across   │                     │
  │  │ CPU: 45% │  │ ERROR:   │  │ services │                     │
  │  │ QPS: 500 │  │ timeout  │  │ App→DB→  │                     │
  │  │ Latency: │  │ on query │  │ Cache→   │                     │
  │  │ 12ms p99 │  │ XYZ...   │  │ response │                     │
  │  └──────────┘  └──────────┘  └──────────┘                     │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 🔑 The Critical Metrics — What to Monitor

### The Database Monitoring Pyramid

```
                    DATABASE MONITORING PYRAMID

            ┌────────────────────────────────────┐
            │         BUSINESS METRICS           │  Revenue impact,
            │         (Top Priority)             │  user experience
            ├────────────────────────────────────┤
            │       APPLICATION METRICS          │  Query latency,
            │       (Query Performance)          │  error rates, QPS
            ├────────────────────────────────────┤
            │       DATABASE INTERNALS           │  Cache hit ratio,
            │       (Engine Health)              │  locks, replication lag
            ├────────────────────────────────────┤
            │       SYSTEM / OS METRICS          │  CPU, RAM, Disk I/O,
            │       (Infrastructure)             │  Network
            └────────────────────────────────────┘
```

### The Must-Monitor Metrics (Universal — All Databases)

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  METRIC CATEGORY       │ WHAT TO WATCH       │ ALERT IF         │
  ├────────────────────────┼─────────────────────┼──────────────────┤
  │                        │                     │                  │
  │  🔴 AVAILABILITY       │ Is DB up?           │ Down > 30s       │
  │                        │ Replication running? │ Replication stops│
  │                        │ Can queries execute? │ Errors > 1%      │
  │                        │                     │                  │
  │  ⚡ PERFORMANCE        │ Query latency (p50, │ p99 > 500ms      │
  │                        │   p95, p99)         │ p50 > 100ms      │
  │                        │ Queries per second   │ Drop > 50%       │
  │                        │ Slow queries count   │ > 10/minute      │
  │                        │ Active connections   │ > 80% of max     │
  │                        │                     │                  │
  │  💾 RESOURCES          │ CPU utilization      │ > 80% sustained  │
  │                        │ Memory usage         │ > 90%            │
  │                        │ Disk usage           │ > 85%            │
  │                        │ Disk I/O (IOPS)      │ > 80% of limit   │
  │                        │ Network throughput   │ > 80% of limit   │
  │                        │                     │                  │
  │  🔄 REPLICATION        │ Replication lag      │ > 30 seconds     │
  │                        │ Replica status       │ Not replicating  │
  │                        │ WAL/Binlog lag       │ > 100MB behind   │
  │                        │                     │                  │
  │  🗄️ CACHE              │ Buffer cache hit %   │ < 95%            │
  │                        │ Query cache hit %    │ < 80%            │
  │                        │ Shared buffers used  │ > 90%            │
  │                        │                     │                  │
  │  🔒 LOCKS              │ Lock wait time       │ > 5 seconds      │
  │                        │ Deadlocks/minute     │ > 0              │
  │                        │ Long-running queries │ > 60 seconds     │
  │                        │                     │                  │
  │  📈 GROWTH             │ Table size growth     │ > 10% per week   │
  │                        │ Row count growth      │ Unusual spikes   │
  │                        │ Index bloat          │ > 30% bloat      │
  │                        │ Transaction ID wrap  │ Approaching limit│
  │                        │                     │                  │
  └────────────────────────┴─────────────────────┴──────────────────┘
```

---

## 🛠️ The Monitoring Stack — Prometheus + Grafana

### Architecture

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    MONITORING STACK                              │
  │                                                                  │
  │  ┌──────────┐     ┌────────────┐     ┌──────────────────────┐  │
  │  │ Database │     │ Exporter   │     │    Prometheus        │  │
  │  │          │────►│ (metrics   │────►│    (time-series DB)  │  │
  │  │ PG/MySQL │     │  endpoint) │     │    scrapes every 15s │  │
  │  │ /Mongo   │     │            │     │                      │  │
  │  └──────────┘     └────────────┘     └──────────┬───────────┘  │
  │                                                  │              │
  │                                            ┌─────▼─────┐       │
  │                                            │  Grafana   │       │
  │                                            │ (dashboards│       │
  │  ┌──────────┐     ┌────────────┐          │  + alerts) │       │
  │  │ App      │────►│ App Metrics│──────────►│            │       │
  │  │ Server   │     │ (custom)   │          │            │       │
  │  └──────────┘     └────────────┘          └─────┬──────┘       │
  │                                                  │              │
  │                                            ┌─────▼─────┐       │
  │                                            │ Alerting   │       │
  │                                            │            │       │
  │                                            │ • PagerDuty│       │
  │                                            │ • Slack    │       │
  │                                            │ • Email    │       │
  │                                            │ • OpsGenie │       │
  │                                            └────────────┘       │
  └──────────────────────────────────────────────────────────────────┘
```

### Database Exporters (Metrics Collectors)

| Database | Exporter | Metrics Port | Install |
|----------|----------|-------------|---------|
| **PostgreSQL** | postgres_exporter | :9187 | `docker run prometheuscommunity/postgres-exporter` |
| **MySQL** | mysqld_exporter | :9104 | `docker run prom/mysqld-exporter` |
| **MongoDB** | mongodb_exporter | :9216 | `docker run percona/mongodb_exporter` |
| **Redis** | redis_exporter | :9121 | `docker run oliver006/redis_exporter` |
| **SQL Server** | sql_exporter | :9399 | Custom or `burningalchemist/sql_exporter` |
| **Oracle** | oracledb_exporter | :9161 | `docker run iamseth/oracledb_exporter` |
| **Elasticsearch** | elasticsearch_exporter | :9114 | `docker run prometheuscommunity/elasticsearch-exporter` |
| **Node (OS)** | node_exporter | :9100 | CPU, RAM, Disk, Network metrics |

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s        # How often to scrape metrics
  evaluation_interval: 15s    # How often to evaluate alerting rules

rule_files:
  - "alerts/database_alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # PostgreSQL metrics
  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']
        labels:
          instance: 'db-primary'
          environment: 'production'

  # MySQL metrics
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysqld-exporter:9104']
        labels:
          instance: 'mysql-primary'

  # MongoDB metrics
  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']

  # OS-level metrics (CPU, RAM, Disk)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### Alert Rules (Production-Ready)

```yaml
# alerts/database_alerts.yml
groups:
  - name: database_alerts
    rules:
      # ======== AVAILABILITY ========
      - alert: DatabaseDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is DOWN on {{ $labels.instance }}"
          description: "Database has been unreachable for more than 1 minute."

      # ======== CONNECTIONS ========
      - alert: TooManyConnections
        expr: pg_stat_activity_count / pg_settings_max_connections * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connection usage > 80% on {{ $labels.instance }}"
          description: "{{ $value }}% of max_connections in use."

      - alert: ConnectionPoolExhausted
        expr: pg_stat_activity_count >= pg_settings_max_connections - 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Almost at max_connections on {{ $labels.instance }}"

      # ======== PERFORMANCE ========
      - alert: SlowQueries
        expr: rate(pg_stat_user_tables_seq_scan[5m]) > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High sequential scan rate on {{ $labels.instance }}"
          description: "Too many sequential scans — missing indexes?"

      - alert: HighQueryLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{handler="/api"}[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p99 query latency > 500ms"

      # ======== DISK ========
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/data"} / node_filesystem_size_bytes{mountpoint="/data"}) * 100 < 15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space < 15% on {{ $labels.instance }}"
          description: "Database disk is almost full! Current: {{ $value }}% free"

      - alert: DiskSpaceWarning
        expr: (node_filesystem_avail_bytes{mountpoint="/data"} / node_filesystem_size_bytes{mountpoint="/data"}) * 100 < 25
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space < 25% on {{ $labels.instance }}"

      # ======== REPLICATION ========
      - alert: ReplicationLag
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag > 30s on {{ $labels.instance }}"
          description: "Current lag: {{ $value }} seconds"

      - alert: ReplicationStopped
        expr: pg_replication_is_replica == 1 and pg_replication_lag < 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Replication has STOPPED on {{ $labels.instance }}"

      # ======== CACHE ========
      - alert: LowCacheHitRatio
        expr: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read) * 100 < 95
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Buffer cache hit ratio < 95% on {{ $labels.instance }}"
          description: "Current hit ratio: {{ $value }}%. Consider increasing shared_buffers."

      # ======== DEADLOCKS ========
      - alert: DeadlocksDetected
        expr: rate(pg_stat_database_deadlocks[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Deadlocks detected on {{ $labels.instance }}"

      # ======== TRANSACTION ID WRAPAROUND (PostgreSQL specific) ========
      - alert: TransactionIdWraparound
        expr: pg_database_age > 1500000000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "TXID wraparound approaching on {{ $labels.instance }}!"
          description: "Age: {{ $value }}. VACUUM immediately!"
```

---

## 🗄️ Database-Specific Monitoring

### PostgreSQL — Built-in Monitoring

```sql
-- ═══════════════════════════════════════════════════
-- pg_stat_statements — THE #1 performance tool ⭐
-- (Must enable: shared_preload_libraries = 'pg_stat_statements')
-- ═══════════════════════════════════════════════════

-- Top 10 slowest queries (by total time)
SELECT
    LEFT(query, 80) AS query_preview,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_time_ms,
    ROUND(mean_exec_time::numeric, 2) AS avg_time_ms,
    ROUND(min_exec_time::numeric, 2) AS min_time_ms,
    ROUND(max_exec_time::numeric, 2) AS max_time_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top 10 most called queries
SELECT
    LEFT(query, 80) AS query_preview,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    rows / GREATEST(calls, 1) AS avg_rows
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Queries with worst cache hit ratio (doing most disk I/O)
SELECT
    LEFT(query, 80) AS query_preview,
    calls,
    shared_blks_hit,
    shared_blks_read,
    ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 100
ORDER BY cache_hit_pct ASC
LIMIT 10;

-- ═══════════════════════════════════════════════════
-- pg_stat_activity — Live query monitoring
-- ═══════════════════════════════════════════════════

-- Currently running queries
SELECT
    pid,
    usename,
    client_addr,
    state,
    LEFT(query, 60) AS query,
    now() - query_start AS duration,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state = 'active'
  AND pid != pg_backend_pid()
ORDER BY duration DESC;

-- Blocked queries (waiting for locks)
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.state = 'active';

-- ═══════════════════════════════════════════════════
-- Table & Index Health
-- ═══════════════════════════════════════════════════

-- Table sizes and row estimates
SELECT
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(pg_table_size(schemaname || '.' || tablename)) AS data_size,
    pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS index_size,
    n_live_tup AS estimated_rows
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- Unused indexes (wasting disk space + slowing writes!)
SELECT
    schemaname || '.' || indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used,
    idx_tup_read AS rows_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND pg_relation_size(indexrelid) > 1024 * 1024  -- > 1MB
ORDER BY pg_relation_size(indexrelid) DESC;

-- Table bloat (dead rows not yet vacuumed)
SELECT
    schemaname || '.' || relname AS table_name,
    n_dead_tup AS dead_rows,
    n_live_tup AS live_rows,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Cache hit ratio (should be > 99%)
SELECT
    datname,
    ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();
```

---

### MySQL — Built-in Monitoring

```sql
-- ═══════════════════════════════════════════════════
-- Slow Query Log (enable in my.cnf)
-- ═══════════════════════════════════════════════════
-- slow_query_log = 1
-- slow_query_log_file = /var/log/mysql/slow.log
-- long_query_time = 1          -- Log queries > 1 second
-- log_queries_not_using_indexes = 1  -- Log queries without indexes

-- ═══════════════════════════════════════════════════
-- Performance Schema — Modern MySQL Monitoring
-- ═══════════════════════════════════════════════════

-- Top 10 queries by total execution time
SELECT
    LEFT(DIGEST_TEXT, 80) AS query_digest,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_time_sec,
    ROUND(AVG_TIMER_WAIT / 1e12, 3) AS avg_time_sec,
    SUM_ROWS_EXAMINED AS rows_examined,
    SUM_ROWS_SENT AS rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Queries doing full table scans
SELECT
    LEFT(DIGEST_TEXT, 80) AS query_digest,
    COUNT_STAR AS executions,
    SUM_NO_INDEX_USED AS no_index_used,
    SUM_ROWS_EXAMINED AS rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
ORDER BY SUM_ROWS_EXAMINED DESC
LIMIT 10;

-- InnoDB Buffer Pool hit ratio
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads')
        /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_pct;

-- Connection statistics
SHOW STATUS LIKE 'Threads%';
SHOW STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';

-- InnoDB status (locks, deadlocks, buffer pool)
SHOW ENGINE INNODB STATUS\G
```

---

### SQL Server — Built-in Monitoring

```sql
-- ═══════════════════════════════════════════════════
-- Query Store (SQL Server 2016+) — Built-in Query Performance Insight
-- ═══════════════════════════════════════════════════

-- Enable Query Store
ALTER DATABASE MyApp SET QUERY_STORE = ON;
ALTER DATABASE MyApp SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 30,
    MAX_STORAGE_SIZE_MB = 1024
);

-- Top 10 most resource-consuming queries
SELECT TOP 10
    qt.query_sql_text,
    rs.count_executions,
    rs.avg_duration / 1000 AS avg_duration_ms,
    rs.avg_cpu_time / 1000 AS avg_cpu_ms,
    rs.avg_logical_io_reads,
    rs.avg_physical_io_reads
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;

-- ═══════════════════════════════════════════════════
-- DMVs — Dynamic Management Views
-- ═══════════════════════════════════════════════════

-- Currently running queries
SELECT
    r.session_id,
    r.status,
    r.command,
    t.text AS query_text,
    r.wait_type,
    r.wait_time,
    r.cpu_time,
    r.total_elapsed_time / 1000 AS elapsed_sec,
    r.reads,
    r.writes
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50
ORDER BY r.total_elapsed_time DESC;

-- Wait statistics (what is SQL Server waiting on?)
SELECT TOP 10
    wait_type,
    wait_time_ms / 1000.0 AS wait_time_sec,
    signal_wait_time_ms / 1000.0 AS signal_wait_sec,
    waiting_tasks_count
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'CLR_SEMAPHORE', 'LAZYWRITER_SLEEP', 'RESOURCE_QUEUE',
    'SLEEP_TASK', 'SLEEP_SYSTEMTASK', 'WAITFOR'
)
ORDER BY wait_time_ms DESC;

-- Missing indexes (SQL Server suggests them!)
SELECT TOP 20
    d.statement AS table_name,
    d.equality_columns,
    d.inequality_columns,
    d.included_columns,
    s.avg_user_impact AS estimated_improvement_pct,
    s.user_seeks + s.user_scans AS potential_uses
FROM sys.dm_db_missing_index_details d
JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
ORDER BY s.avg_user_impact * (s.user_seeks + s.user_scans) DESC;

-- Buffer cache hit ratio
SELECT
    (CAST(a.cntr_value AS FLOAT) / CAST(b.cntr_value AS FLOAT)) * 100.0 AS cache_hit_pct
FROM sys.dm_os_performance_counters a
JOIN sys.dm_os_performance_counters b ON a.object_name = b.object_name
WHERE a.counter_name = 'Buffer cache hit ratio'
  AND b.counter_name = 'Buffer cache hit ratio base';
```

---

### MongoDB — Built-in Monitoring

```javascript
// ═══════════════════════════════════════════════════
// MongoDB Monitoring Commands
// ═══════════════════════════════════════════════════

// Server status (comprehensive overview)
db.serverStatus()

// Key sections of serverStatus:
db.serverStatus().connections      // Active connections
db.serverStatus().opcounters       // Insert/query/update/delete counts
db.serverStatus().mem              // Memory usage
db.serverStatus().wiredTiger       // Storage engine stats
db.serverStatus().globalLock       // Lock statistics

// Current operations (equivalent to SHOW PROCESSLIST)
db.currentOp({ "active": true, "secs_running": { "$gt": 5 } })

// Kill a long-running operation
db.killOp(opId)

// Collection statistics
db.orders.stats()
// Shows: document count, storage size, index sizes, average doc size

// Index usage statistics
db.orders.aggregate([{ $indexStats: {} }])

// Database profiler (log slow queries)
db.setProfilingLevel(1, { slowms: 100 })  // Log queries > 100ms

// View profiled queries
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()

// Explain a query (execution plan)
db.orders.find({ status: "pending" }).explain("executionStats")
// Check: totalDocsExamined vs nReturned
// If totalDocsExamined >> nReturned → missing index!

// Replica set status
rs.status()       // Replication health
rs.printReplicationInfo()  // Oplog window
rs.printSecondaryReplicationInfo()  // Replica lag

// mongostat (CLI tool — real-time stats)
// mongostat --host localhost:27017 --rowcount 20
// Shows: inserts/queries/updates/deletes per second, connections, memory
```

---

## 📊 Grafana Dashboard Setup

### Essential Dashboard Panels

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DATABASE MONITORING DASHBOARD                                   │
  ├──────────────────┬──────────────────┬───────────────────────────┤
  │                  │                  │                           │
  │   🟢 Status:    │  ⚡ QPS: 2,450  │  ⏱️ Latency p99: 45ms    │
  │   UP (3d 14h)   │  (queries/sec)   │  p95: 12ms | p50: 3ms    │
  │                  │                  │                           │
  ├──────────────────┴──────────────────┴───────────────────────────┤
  │                                                                  │
  │  📈 Query Throughput (last 24h)                                  │
  │  ▁▂▃▃▃▃▅▇█████▇▇▅▅▃▃▂▂▁▁                                      │
  │  00  04  08  12  16  20  24                                      │
  │                                                                  │
  ├──────────────────────────────────────────────────────────────────┤
  │                                                                  │
  │  ⏱️ Query Latency Distribution (p50/p95/p99)                     │
  │  ▁▂▂▃▃▃▃▃▃▂▂▂▂▃▃▃▃▂▂▁▁▁                                        │
  │  p99: ────  p95: ────  p50: ────                                │
  │                                                                  │
  ├───────────────────┬────────────────────────────────────────────┤
  │                   │                                            │
  │  💾 CPU: 34%     │  🗄️ Connections:                           │
  │  ▇▇▇▇░░░░░░      │  Active: 12 / Max: 100                    │
  │                   │  Idle: 38                                   │
  │  💿 Disk: 67%    │  Waiting: 0 ✅                              │
  │  ▇▇▇▇▇▇▇░░░      │                                            │
  │                   │                                            │
  │  🧠 RAM: 72%     │  🔄 Replication Lag:                       │
  │  ▇▇▇▇▇▇▇░░░      │  0.5 seconds ✅                            │
  │                   │                                            │
  ├───────────────────┴────────────────────────────────────────────┤
  │                                                                  │
  │  🐌 Top 5 Slowest Queries (last 1h)                             │
  │  ┌──────────────────────────────────────┬───────┬─────────────┐ │
  │  │ Query                                │ Calls │ Avg Time    │ │
  │  ├──────────────────────────────────────┼───────┼─────────────┤ │
  │  │ SELECT * FROM orders WHERE status... │ 1,204 │ 234ms ⚠️    │ │
  │  │ UPDATE inventory SET quantity...     │   456 │ 189ms       │ │
  │  │ SELECT u.* FROM users u JOIN...      │ 8,901 │  45ms       │ │
  │  │ INSERT INTO audit_log...             │12,340 │  12ms       │ │
  │  │ SELECT COUNT(*) FROM products...     │ 2,100 │   8ms       │ │
  │  └──────────────────────────────────────┴───────┴─────────────┘ │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

### Docker Compose — Complete Monitoring Stack

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts:/etc/prometheus/alerts
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:${MONITOR_PASSWORD}@db-primary:5432/myapp?sslmode=disable"
    ports:
      - "9187:9187"

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml

volumes:
  prometheus_data:
  grafana_data:
```

---

## 🔍 Slow Query Analysis — A Systematic Approach

### The Slow Query Investigation Workflow

```
  🐌 Alert: "p99 latency > 500ms"
       │
       ▼
  Step 1: IDENTIFY the slow query
       │  → pg_stat_statements / slow query log / APM tool
       │
       ▼
  Step 2: EXPLAIN the query
       │  → EXPLAIN (ANALYZE, BUFFERS) SELECT ...
       │  → Look for: Seq Scan, Nested Loop, high row estimates
       │
       ▼
  Step 3: CHECK indexes
       │  → Does the query use the right index?
       │  → Is the index selective enough?
       │  → Is a composite index needed?
       │
       ▼
  Step 4: CHECK statistics
       │  → Are table statistics up to date?
       │  → ANALYZE table_name;
       │
       ▼
  Step 5: CHECK for bloat
       │  → Table bloat → VACUUM
       │  → Index bloat → REINDEX
       │
       ▼
  Step 6: OPTIMIZE the query
       │  → Rewrite JOINs, add CTEs, use covering index
       │  → Consider query hints (last resort)
       │
       ▼
  Step 7: VERIFY the fix
       │  → Re-run EXPLAIN ANALYZE
       │  → Monitor p99 latency
       │  → Confirm improvement
       │
       ▼
  ✅ Problem resolved!
```

```sql
-- PostgreSQL: Complete slow query investigation

-- Step 1: Find the slow query
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 5;

-- Step 2: EXPLAIN ANALYZE (run the query and show execution plan)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.username
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
  AND o.created_at > '2026-01-01';

-- Look for these RED FLAGS in the plan:
--   ⚠️ Seq Scan on large table (missing index)
--   ⚠️ Nested Loop with high row count (consider Hash Join)
--   ⚠️ Actual rows >> estimated rows (stale statistics)
--   ⚠️ Buffers: read >> shared hit (data not in cache)

-- Step 3: Check index usage
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE relname = 'orders';

-- Step 4: Update statistics
ANALYZE orders;

-- Step 5: Check bloat
SELECT n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- Step 6: Create the right index
CREATE INDEX CONCURRENTLY idx_orders_status_created
ON orders(status, created_at)
WHERE status = 'pending';  -- Partial index for common filter!

-- Step 7: Verify improvement
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, u.username
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
  AND o.created_at > '2026-01-01';
-- Should now show Index Scan instead of Seq Scan ✅
```

---

## 📋 Monitoring Checklist by Database

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  PostgreSQL Monitoring Checklist:                                │
  │  ✅ pg_stat_statements enabled                                   │
  │  ✅ pg_stat_activity monitored (connections, queries)            │
  │  ✅ Buffer cache hit ratio > 99%                                │
  │  ✅ Autovacuum running regularly                                 │
  │  ✅ Replication lag < 5 seconds                                  │
  │  ✅ TXID wraparound monitored                                   │
  │  ✅ Unused indexes identified and dropped                       │
  │  ✅ Table/index bloat tracked                                   │
  ├──────────────────────────────────────────────────────────────────┤
  │  MySQL Monitoring Checklist:                                     │
  │  ✅ Slow query log enabled (long_query_time = 1)                │
  │  ✅ Performance Schema enabled                                  │
  │  ✅ InnoDB buffer pool hit ratio > 99%                          │
  │  ✅ Threads_running vs Threads_connected                        │
  │  ✅ Replication lag (Seconds_Behind_Master)                     │
  │  ✅ InnoDB deadlock monitor                                     │
  │  ✅ Binary log disk usage                                       │
  ├──────────────────────────────────────────────────────────────────┤
  │  SQL Server Monitoring Checklist:                                │
  │  ✅ Query Store enabled                                         │
  │  ✅ Wait statistics analyzed                                    │
  │  ✅ Missing index DMV checked                                   │
  │  ✅ Buffer cache hit ratio > 99%                                │
  │  ✅ SQL Agent jobs monitored                                    │
  │  ✅ AG (Always On) health                                      │
  │  ✅ TempDB contention                                           │
  ├──────────────────────────────────────────────────────────────────┤
  │  MongoDB Monitoring Checklist:                                   │
  │  ✅ db.currentOp() for long operations                          │
  │  ✅ Profiler enabled for slow queries (slowms: 100)             │
  │  ✅ WiredTiger cache utilization                                │
  │  ✅ Oplog window size                                           │
  │  ✅ Replica set member states                                   │
  │  ✅ Collection scan ratio (scans vs index hits)                 │
  │  ✅ Connection count                                            │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 🏢 Commercial Monitoring Tools

| Tool | Type | Best For | Key Feature |
|------|------|----------|-------------|
| **Datadog** | Full-stack APM | All databases + apps | Deep integration, AI anomaly detection |
| **New Relic** | Full-stack APM | Full-stack observability | Query analysis, distributed tracing |
| **Percona PMM** | Database-specific | MySQL, PostgreSQL, MongoDB | Open source, Query Analytics |
| **pganalyze** | PostgreSQL only | PG deep monitoring | EXPLAIN plan analysis, index advisor |
| **SolarWinds DPA** | SQL Server/Oracle | Enterprise DBAs | Wait-time analysis, historical trends |
| **Oracle Enterprise Manager** | Oracle | Oracle ecosystem | Complete Oracle management |
| **AWS CloudWatch** | AWS services | RDS/Aurora | Native integration, Performance Insights |
| **Azure Monitor** | Azure services | Azure SQL | Built-in intelligence, auto-tuning |
| **VividCortex (now SolarWinds)** | Multi-database | SaaS monitoring | Real-time query analysis |

---

## 🏆 Key Takeaways

| Concept | Remember This |
|---------|---------------|
| **Monitor ≠ Observe** | Monitoring = known metrics; Observability = ask any question |
| **Three Pillars** | Metrics + Logs + Traces = full picture |
| **Prometheus + Grafana** | Industry standard, open source, works with everything |
| **pg_stat_statements** | PostgreSQL's #1 performance tool — always enable it |
| **Slow query log** | MySQL's equivalent — set long_query_time = 1 |
| **Query Store** | SQL Server 2016+ built-in query performance tracker |
| **Buffer cache hit** | Should be > 99% — if not, increase shared_buffers/buffer_pool |
| **Alert on what matters** | Disk > 85%, connections > 80%, latency p99 > 500ms |
| **Don't alert on noise** | CPU spikes are normal; sustained high CPU is a problem |
| **Slow query workflow** | Identify → EXPLAIN → Index → Statistics → Verify |

---

> 🎉 **Congratulations!** You've completed **PART 5 — Data Engineering & Database Tooling!**
> You now understand ETL/ELT pipelines, database migrations, ORMs, connection pooling, and monitoring.
>
> 🚀 **Next Up:** [Part 6 — Data Warehousing & Analytics](../17-Data-Warehouse/01-DW-Concepts.md) — Dive into OLAP, Star Schemas, Snowflake, BigQuery, and Redshift.

---

*Last Updated: June 2026*
