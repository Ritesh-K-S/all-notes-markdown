# 🛡️ Chapter 2C.6 — SQL Server Always On & High Availability

> **Level:** 🔴 Advanced | 🔥 High Demand
> **Time to Master:** ~5-6 hours
> **Prerequisites:** Chapter 2C.1 (SQL Server Architecture), Chapter 2C.4 (Performance Tuning)

> **"A database that goes down is like a hospital that closes its emergency room — unacceptable."**

---

## 📌 What You'll Master

By the end of this chapter, you will:
- Understand **why** High Availability (HA) and Disaster Recovery (DR) are different things
- Architect SQL Server solutions with **99.99%+ uptime**
- Configure **Always On Availability Groups** — the gold standard
- Know **every HA option** SQL Server offers and when to use each
- Design **failover strategies** that don't lose a single transaction
- Answer HA/DR interview questions like a **senior DBA**

---

## 🎯 The Big Picture — HA vs DR

> *"It's 3 AM. Your primary SQL Server just died. 10,000 users are getting errors. Your phone is ringing. What's your plan?"*

If you don't have one — you're about to have a **very bad night**.

```
╔══════════════════════════════════════════════════════════════════╗
║              HIGH AVAILABILITY vs DISASTER RECOVERY              ║
╠══════════════════════════════╦═══════════════════════════════════╣
║  HIGH AVAILABILITY (HA)      ║  DISASTER RECOVERY (DR)          ║
╠══════════════════════════════╬═══════════════════════════════════╣
║                              ║                                   ║
║  "Keep running during        ║  "Get back online after a         ║
║   small failures"            ║   catastrophic failure"           ║
║                              ║                                   ║
║  Server crash, disk failure, ║  Data center fire, flood,         ║
║  network blip, OS update     ║  ransomware, region-wide outage  ║
║                              ║                                   ║
║  Automatic failover          ║  Manual or automated failover    ║
║  (seconds to minutes)        ║  (minutes to hours)              ║
║                              ║                                   ║
║  Same data center / rack     ║  Different city / region /       ║
║                              ║  continent                       ║
║                              ║                                   ║
║  Goal: ZERO downtime         ║  Goal: Minimize data loss        ║
║  (RTO ≈ 0)                   ║  (RPO ≈ 0 to minutes)           ║
║                              ║                                   ║
╠══════════════════════════════╬═══════════════════════════════════╣
║  Technologies:               ║  Technologies:                    ║
║  • Always On AG (sync)       ║  • Always On AG (async)          ║
║  • Failover Clustering       ║  • Log Shipping                  ║
║  • Database Mirroring        ║  • Backup/Restore                ║
║                              ║  • Azure Site Recovery           ║
╚══════════════════════════════╩═══════════════════════════════════╝
```

### Key Metrics — RTO & RPO

```
                     DISASTER HAPPENS HERE
                              │
  ◄───── RPO ─────►          │          ◄───── RTO ─────►
  (Recovery Point             │          (Recovery Time
   Objective)                 │           Objective)
                              │
  "How much data              │          "How long until
   can we LOSE?"              │           we're back UP?"
                              │
  ──────────────────────────── ┼ ────────────────────────────
  Last backup/sync     💥 Failure      System restored
  
  
  RPO = 0     → Zero data loss (synchronous replication)
  RPO = 15min → Lose at most 15 minutes of transactions
  RPO = 24hr  → Lose at most 1 day (daily backup only)
  
  RTO = 0     → Instant failover (automatic)
  RTO = 5min  → Back online in 5 minutes
  RTO = 4hr   → Restore from backup (acceptable for some systems)
```

> 💡 **The Cost Equation**: Lower RPO/RTO = Higher cost. **99.9%** uptime = ~8.7 hours down/year. **99.99%** = ~52 minutes/year. **99.999%** = ~5 minutes/year ($$$$).

---

## 🏗️ SQL Server HA/DR Options — Complete Overview

```
╔══════════════════════════════════════════════════════════════════════╗
║                SQL SERVER HA/DR TECHNOLOGIES                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  🥇 ALWAYS ON AVAILABILITY GROUPS (SQL 2012+)              │    ║
║  │     THE gold standard. Group of databases that fail over   │    ║
║  │     together. Sync + Async replicas. Readable secondaries. │    ║
║  │     RPO: 0 (sync) | RTO: seconds (automatic failover)     │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  🥈 ALWAYS ON FAILOVER CLUSTER INSTANCES (FCI)             │    ║
║  │     Shared storage cluster. Instance-level failover.       │    ║
║  │     Good for: protecting against server hardware failure.  │    ║
║  │     RPO: 0 | RTO: 30 seconds - few minutes                │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  🥉 LOG SHIPPING (Classic, Simple, Reliable)               │    ║
║  │     Backup → Copy → Restore transaction logs to standby.   │    ║
║  │     Low-tech but battle-tested for DR.                     │    ║
║  │     RPO: minutes (depends on schedule) | RTO: minutes      │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  ⚠️ DATABASE MIRRORING (Deprecated in SQL 2012+)           │    ║
║  │     One-to-one database replication.                       │    ║
║  │     Replaced by Always On AG. Don't use for new projects.  │    ║
║  │     RPO: 0 (sync) | RTO: seconds (with witness)           │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  📸 BACKUP & RESTORE (The Last Resort)                     │    ║
║  │     Full + Differential + Transaction Log backups.         │    ║
║  │     Every SQL Server deployment MUST have this.            │    ║
║  │     RPO: depends on backup frequency | RTO: minutes-hours  │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🥇 Always On Availability Groups (AG) — The Deep Dive

### What Is an Availability Group?

> An Availability Group is a **group of databases** that fail over **together** as a single unit. One primary replica serves reads and writes. One or more secondary replicas receive changes and can serve read-only queries.

```
╔══════════════════════════════════════════════════════════════════╗
║              ALWAYS ON AVAILABILITY GROUP                        ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │                AVAILABILITY GROUP: "AG_Sales"           │    ║
║  │                                                         │    ║
║  │  Contains: SalesDB, InventoryDB, CustomerDB            │    ║
║  │  (All fail over TOGETHER — they're a logical unit)      │    ║
║  │                                                         │    ║
║  │  ┌───────────────┐                                      │    ║
║  │  │  LISTENER     │  ← Virtual Network Name              │    ║
║  │  │  ag_sales_lstn│     Apps connect HERE (not to nodes) │    ║
║  │  │  10.0.1.100   │     Automatically routes to primary  │    ║
║  │  └───────┬───────┘                                      │    ║
║  │          │                                              │    ║
║  │  ┌───────▼───────┐  ┌───────────────┐  ┌─────────────┐│    ║
║  │  │   PRIMARY     │  │  SECONDARY    │  │  SECONDARY  ││    ║
║  │  │   (Node 1)    │  │  (Node 2)     │  │  (Node 3)   ││    ║
║  │  │               │  │               │  │             ││    ║
║  │  │  Read + Write │  │  Read-Only    │  │  Read-Only  ││    ║
║  │  │  ✅ All ops   │  │  ✅ Reports   │  │  ✅ Backups ││    ║
║  │  │               │  │               │  │             ││    ║
║  │  │  ──sync──►    │  │  ◄──async──   │  │             ││    ║
║  │  │  (same DC)    │  │  (remote DC)  │  │  (DR site)  ││    ║
║  │  └───────────────┘  └───────────────┘  └─────────────┘│    ║
║  │                                                         │    ║
║  └─────────────────────────────────────────────────────────┘    ║
║                                                                  ║
║  WSFC (Windows Server Failover Clustering) manages the nodes    ║
╚══════════════════════════════════════════════════════════════════╝
```

---

### Synchronous vs Asynchronous Commit

```
SYNCHRONOUS COMMIT (Zero Data Loss):
═══════════════════════════════════════

  App writes          Primary              Secondary
  ─────────          ─────────            ──────────
      │                  │                     │
   1. │──INSERT──►       │                     │
      │                  │                     │
   2. │            Write to log                │
      │                  │                     │
   3. │                  │──── Send log ──────►│
      │                  │                     │
   4. │                  │     Write to log    │
      │                  │     (harden)        │
   5. │                  │◄── ACK (hardened) ──│
      │                  │                     │
   6. │◄── COMMIT OK ───│                     │
      │                  │                     │

  ✅ ZERO data loss — both nodes have the transaction
  ❌ SLOWER — must wait for secondary to confirm
  📍 USE FOR: same data center, low latency (<1ms)


ASYNCHRONOUS COMMIT (Possible Data Loss):
═════════════════════════════════════════

  App writes          Primary              Secondary
  ─────────          ─────────            ──────────
      │                  │                     │
   1. │──INSERT──►       │                     │
      │                  │                     │
   2. │            Write to log                │
      │                  │                     │
   3. │◄── COMMIT OK ───│                     │  ← Returns IMMEDIATELY!
      │                  │                     │
   4. │                  │──── Send log ──────►│  ← Happens in background
      │                  │                     │
      │                  │     Write to log    │
      │                  │                     │

  ❌ POSSIBLE data loss (transactions in transit during failure)
  ✅ FASTER — no waiting
  📍 USE FOR: cross-data-center (high latency), DR replicas
```

> 💡 **Key Decision**: Same city / same DC → **Synchronous**. Different city / cross-region → **Asynchronous**. Never use sync across WAN (it will kill your performance).

---

### Automatic vs Manual Failover

```
╔══════════════════════════════════════════════════════════════════╗
║                    FAILOVER MODES                                ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  AUTOMATIC FAILOVER (Requires sync commit + WSFC health check)  ║
║  ┌───────────────────────────────────────────────────────────┐  ║
║  │  Primary dies → WSFC detects in ~10 seconds               │  ║
║  │  → Secondary promoted to Primary automatically            │  ║
║  │  → Listener redirects connections to new Primary          │  ║
║  │  → Applications reconnect (brief connection reset)        │  ║
║  │                                                           │  ║
║  │  Total downtime: ~10-30 seconds                           │  ║
║  │  Data loss: ZERO (sync commit guarantees it)              │  ║
║  │                                                           │  ║
║  │  Requirements:                                            │  ║
║  │  ✅ Synchronous commit mode                               │  ║
║  │  ✅ Both replicas SYNCHRONIZED state                      │  ║
║  │  ✅ WSFC quorum is healthy                                │  ║
║  └───────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║  MANUAL FAILOVER (Planned — zero data loss)                     ║
║  ┌───────────────────────────────────────────────────────────┐  ║
║  │  DBA initiates failover (maintenance window, patching)    │  ║
║  │  → Waits for all transactions to sync                     │  ║
║  │  → Switches roles cleanly                                 │  ║
║  │  → Zero data loss                                         │  ║
║  │                                                           │  ║
║  │  Use for: OS patching, SQL Server upgrades, hardware      │  ║
║  │  maintenance — the "rolling upgrade" pattern              │  ║
║  └───────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║  FORCED FAILOVER (Emergency — possible data loss!)              ║
║  ┌───────────────────────────────────────────────────────────┐  ║
║  │  Primary is DEAD and won't come back                      │  ║
║  │  Async secondary may not have all transactions            │  ║
║  │  → DBA forces promotion of secondary                      │  ║
║  │  → Any un-synced transactions are LOST                    │  ║
║  │                                                           │  ║
║  │  ⚠️ LAST RESORT — only when primary is unrecoverable     │  ║
║  │  ⚠️ Must reconcile data after primary recovers           │  ║
║  └───────────────────────────────────────────────────────────┘  ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

### Setting Up Always On AG — Step by Step

```
PREREQUISITES:
══════════════════════════════════════════════════════════
1. Windows Server Failover Clustering (WSFC) installed
2. All nodes joined to the same WSFC cluster
3. SQL Server Enterprise Edition (or Standard with limits)
4. Same SQL Server version on all nodes
5. Database in FULL recovery model
6. Full backup of each database taken
```

```sql
-- STEP 1: Enable Always On on each SQL Server instance
-- (Done via SQL Server Configuration Manager → Properties → Always On tab)
-- Or via PowerShell:
-- Enable-SqlAlwaysOn -ServerInstance "Node1\SQLInstance" -Force

-- STEP 2: Take a full backup of each database
BACKUP DATABASE SalesDB 
TO DISK = '\\SharedPath\SalesDB_Full.bak'
WITH FORMAT, COMPRESSION;

BACKUP LOG SalesDB 
TO DISK = '\\SharedPath\SalesDB_Log.trn'
WITH FORMAT, COMPRESSION;

-- STEP 3: Restore databases on secondary (WITH NORECOVERY!)
-- On Node 2:
RESTORE DATABASE SalesDB 
FROM DISK = '\\SharedPath\SalesDB_Full.bak'
WITH NORECOVERY, MOVE 'SalesDB_Data' TO 'D:\Data\SalesDB.mdf',
     MOVE 'SalesDB_Log' TO 'E:\Logs\SalesDB_ldf';

RESTORE LOG SalesDB 
FROM DISK = '\\SharedPath\SalesDB_Log.trn'
WITH NORECOVERY;

-- STEP 4: Create the Availability Group (on Primary)
CREATE AVAILABILITY GROUP [AG_Sales]
WITH (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY,  -- Backups run on secondary
    DB_FAILOVER = ON,                         -- Database-level health detection
    CLUSTER_TYPE = WSFC                       -- Windows Server Failover Cluster
)
FOR DATABASE SalesDB, InventoryDB, CustomerDB
REPLICA ON
    'Node1' WITH (
        ENDPOINT_URL = 'TCP://Node1.corp.local:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = MANUAL,
        SECONDARY_ROLE (
            ALLOW_CONNECTIONS = READ_ONLY,
            READ_ONLY_ROUTING_URL = 'TCP://Node1.corp.local:1433'
        )
    ),
    'Node2' WITH (
        ENDPOINT_URL = 'TCP://Node2.corp.local:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = MANUAL,
        SECONDARY_ROLE (
            ALLOW_CONNECTIONS = READ_ONLY,
            READ_ONLY_ROUTING_URL = 'TCP://Node2.corp.local:1433'
        )
    ),
    'Node3' WITH (
        ENDPOINT_URL = 'TCP://Node3.corp.local:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,  -- DR replica
        FAILOVER_MODE = MANUAL,                   -- No auto-failover for async
        SEEDING_MODE = MANUAL,
        SECONDARY_ROLE (
            ALLOW_CONNECTIONS = READ_ONLY,
            READ_ONLY_ROUTING_URL = 'TCP://Node3.corp.local:1433'
        )
    );

-- STEP 5: Create the Listener
ALTER AVAILABILITY GROUP [AG_Sales]
ADD LISTENER 'ag_sales_lstn' (
    WITH IP (
        (N'10.0.1.100', N'255.255.255.0'),   -- Subnet 1
        (N'10.0.2.100', N'255.255.255.0')    -- Subnet 2 (multi-subnet)
    ),
    PORT = 1433
);

-- STEP 6: Join secondary replicas to the AG
-- On Node2:
ALTER AVAILABILITY GROUP [AG_Sales] JOIN;
ALTER DATABASE SalesDB SET HADR AVAILABILITY GROUP = [AG_Sales];

-- On Node3:
ALTER AVAILABILITY GROUP [AG_Sales] JOIN;
ALTER DATABASE SalesDB SET HADR AVAILABILITY GROUP = [AG_Sales];
```

---

### Read-Only Routing — Offload Reports to Secondaries

```
╔══════════════════════════════════════════════════════════════════╗
║              READ-ONLY ROUTING                                   ║
║                                                                  ║
║  App connects to LISTENER with ReadOnly intent:                  ║
║  "Server=ag_sales_lstn;Database=SalesDB;                        ║
║   ApplicationIntent=ReadOnly"                                    ║
║                                                                  ║
║  ┌─────────────────┐                                            ║
║  │   LISTENER      │                                            ║
║  │  ag_sales_lstn  │                                            ║
║  └────────┬────────┘                                            ║
║           │                                                      ║
║     ┌─────┴──────┐                                              ║
║     │            │                                              ║
║     ▼            ▼                                              ║
║  ReadWrite    ReadOnly                                          ║
║  Intent       Intent                                            ║
║     │            │                                              ║
║     ▼            ▼                                              ║
║  PRIMARY      SECONDARY                                         ║
║  (Node 1)    (Node 2 or 3)                                     ║
║  Handles:    Handles:                                           ║
║  • INSERT    • Reports                                          ║
║  • UPDATE    • Analytics                                        ║
║  • DELETE    • Backups                                          ║
║  • SELECT    • Read-heavy queries                               ║
║                                                                  ║
║  💡 This DOUBLES your read capacity at ZERO extra cost!         ║
╚══════════════════════════════════════════════════════════════════╝
```

```sql
-- Configure read-only routing
ALTER AVAILABILITY GROUP [AG_Sales]
MODIFY REPLICA ON 'Node1' WITH (
    PRIMARY_ROLE (
        READ_ONLY_ROUTING_LIST = ('Node2', 'Node3')
    )
);

ALTER AVAILABILITY GROUP [AG_Sales]
MODIFY REPLICA ON 'Node2' WITH (
    PRIMARY_ROLE (
        READ_ONLY_ROUTING_LIST = ('Node1', 'Node3')
    )
);
```

---

### Monitoring Always On AG — Essential DMVs

```sql
-- 🔍 Check AG health status
SELECT 
    ag.name AS ag_name,
    ar.replica_server_name,
    ars.role_desc,                        -- PRIMARY or SECONDARY
    ars.connected_state_desc,             -- CONNECTED or DISCONNECTED
    ars.synchronization_health_desc,      -- HEALTHY, NOT_HEALTHY, PARTIALLY_HEALTHY
    ars.operational_state_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;

-- 🔍 Check database synchronization status
SELECT 
    ag.name AS ag_name,
    ar.replica_server_name,
    db.name AS database_name,
    drs.synchronization_state_desc,       -- SYNCHRONIZED, SYNCHRONIZING, NOT SYNCHRONIZING
    drs.synchronization_health_desc,      -- HEALTHY, NOT_HEALTHY
    drs.log_send_queue_size,              -- KB of log waiting to be sent
    drs.log_send_rate,                    -- KB/sec
    drs.redo_queue_size,                  -- KB of log waiting to be redone
    drs.redo_rate,                        -- KB/sec
    drs.last_sent_time,
    drs.last_received_time,
    drs.last_hardened_time,
    drs.last_redone_time,
    -- Estimated recovery time:
    CASE WHEN drs.redo_rate > 0 
         THEN drs.redo_queue_size / drs.redo_rate 
         ELSE 0 
    END AS estimated_redo_seconds
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.availability_groups ag ON ar.group_id = ag.group_id
JOIN sys.databases db ON drs.database_id = db.database_id
ORDER BY ag.name, ar.replica_server_name, db.name;

-- 🔍 Check listener details
SELECT 
    agl.dns_name,
    agl.port,
    aglip.ip_address,
    aglip.ip_subnet_mask,
    aglip.state_desc
FROM sys.availability_group_listeners agl
JOIN sys.availability_group_listener_ip_addresses aglip 
    ON agl.listener_id = aglip.listener_id;

-- ⚠️ ALERT: Find databases falling behind (lag > 5 seconds)
SELECT 
    ar.replica_server_name,
    db.name,
    drs.log_send_queue_size AS send_queue_KB,
    drs.redo_queue_size AS redo_queue_KB,
    DATEDIFF(SECOND, drs.last_redone_time, GETDATE()) AS seconds_behind
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.databases db ON drs.database_id = db.database_id
WHERE drs.is_local = 0  -- Remote replicas only
  AND DATEDIFF(SECOND, drs.last_redone_time, GETDATE()) > 5;
```

---

## 🥈 Failover Cluster Instances (FCI)

### How FCI Works

```
╔══════════════════════════════════════════════════════════════════╗
║           FAILOVER CLUSTER INSTANCE (FCI)                        ║
║                                                                  ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │              SHARED STORAGE (SAN / S2D)                  │   ║
║  │         ┌─────────────────────────────┐                  │   ║
║  │         │  Data files (.mdf, .ndf)    │                  │   ║
║  │         │  Log files (.ldf)           │                  │   ║
║  │         │  TempDB (can be local)      │                  │   ║
║  │         └─────────────┬───────────────┘                  │   ║
║  │                       │                                  │   ║
║  │         ┌─────────────┼───────────────┐                  │   ║
║  │         │             │               │                  │   ║
║  │    ┌────▼────┐   ┌────▼────┐   ┌─────▼────┐            │   ║
║  │    │  Node 1 │   │  Node 2 │   │  Node 3  │            │   ║
║  │    │ (ACTIVE)│   │(PASSIVE)│   │(PASSIVE) │            │   ║
║  │    │  SQL ✅ │   │  SQL ⏸ │   │  SQL ⏸  │            │   ║
║  │    └─────────┘   └─────────┘   └──────────┘            │   ║
║  │         │                                               │   ║
║  │         ▼                                               │   ║
║  │    ┌──────────────────────┐                             │   ║
║  │    │  Virtual Network     │                             │   ║
║  │    │  Name (VNN)          │                             │   ║
║  │    │  "SQLCLUSTER01"      │  ← Apps connect here       │   ║
║  │    │  10.0.1.50           │                             │   ║
║  │    └──────────────────────┘                             │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                                                                  ║
║  On failure: Node 1 dies → SQL starts on Node 2                 ║
║  Same databases, same IP, same name → apps reconnect            ║
╚══════════════════════════════════════════════════════════════════╝
```

### FCI vs AG — Comparison

```
┌────────────────────────┬───────────────────────┬─────────────────────┐
│ Feature                │ FCI                   │ Always On AG        │
├────────────────────────┼───────────────────────┼─────────────────────┤
│ Failover Unit          │ Entire SQL instance   │ Group of databases  │
│ Storage                │ Shared (SAN)          │ Local (each node)   │
│ Readable Secondary     │ ❌ No                │ ✅ Yes              │
│ Per-Database Failover  │ ❌ No                │ ✅ Yes              │
│ Storage Failure        │ ❌ Single point       │ ✅ Each has copy    │
│ Cross-Subnet           │ ✅ Yes               │ ✅ Yes              │
│ Licensing              │ Need 1 license        │ Need per-node       │
│                        │ (passive = free)      │ (passive = free     │
│                        │                       │  in Enterprise)     │
│ Best For               │ Instance-level HA     │ Database-level HA   │
│                        │ + legacy apps         │ + read scale-out    │
├────────────────────────┼───────────────────────┼─────────────────────┤
│ Can be combined?       │ YES! FCI + AG = ultimate protection       │
│                        │ (AG nodes can be FCIs themselves)          │
└────────────────────────┴────────────────────────────────────────────┘
```

---

## 🥉 Log Shipping — The Workhorse DR Solution

### How Log Shipping Works

```
╔══════════════════════════════════════════════════════════════════╗
║                     LOG SHIPPING                                 ║
║                                                                  ║
║  PRIMARY SERVER                      SECONDARY SERVER(S)        ║
║  (Production)                        (Standby / DR)             ║
║                                                                  ║
║  ┌────────────┐                      ┌────────────┐             ║
║  │  SalesDB   │                      │  SalesDB   │             ║
║  │  (Online)  │                      │ (Standby/  │             ║
║  │            │                      │  Read-Only)│             ║
║  └──────┬─────┘                      └──────▲─────┘             ║
║         │                                   │                    ║
║    1. BACKUP JOB                       3. RESTORE JOB           ║
║    (every 15 min)                      (every 15 min)           ║
║         │                                   │                    ║
║         ▼                                   │                    ║
║  ┌──────────────┐    2. COPY JOB     ┌──────┴──────┐           ║
║  │ \\share\logs │───────────────────►│ \\share\logs│           ║
║  │ TLog_001.trn │   (File copy)      │ TLog_001.trn│           ║
║  │ TLog_002.trn │                    │ TLog_002.trn│           ║
║  └──────────────┘                    └─────────────┘           ║
║                                                                  ║
║  THREE SQL AGENT JOBS:                                          ║
║  Job 1 (Primary):   Backup transaction log every N minutes      ║
║  Job 2 (Secondary): Copy .trn files from share to local         ║
║  Job 3 (Secondary): Restore .trn files to standby database      ║
║                                                                  ║
║  RPO = Backup interval (typically 5-15 minutes)                 ║
║  RTO = Time to bring secondary online (minutes)                 ║
╚══════════════════════════════════════════════════════════════════╝
```

```sql
-- Setting up Log Shipping (simplified)

-- ON PRIMARY: Create backup job
BACKUP LOG SalesDB 
TO DISK = '\\FileShare\LogShip\SalesDB_LS_001.trn'
WITH FORMAT, COMPRESSION;

-- ON SECONDARY: Restore with NORECOVERY (initial setup)
RESTORE DATABASE SalesDB 
FROM DISK = '\\FileShare\Backup\SalesDB_Full.bak'
WITH NORECOVERY;

-- Restore transaction logs (repeated by job)
RESTORE LOG SalesDB 
FROM DISK = '\\FileShare\LogShip\SalesDB_LS_001.trn'
WITH NORECOVERY;  -- Keep in NORECOVERY for more logs

-- OR: WITH STANDBY for read-only access between restores
RESTORE LOG SalesDB 
FROM DISK = '\\FileShare\LogShip\SalesDB_LS_001.trn'
WITH STANDBY = 'D:\Standby\SalesDB_Undo.dat';

-- TO FAILOVER (bring secondary online):
RESTORE DATABASE SalesDB WITH RECOVERY;  -- Makes it fully online
```

### When to Use Log Shipping

```
✅ USE LOG SHIPPING WHEN:
• Budget is tight (works on Standard Edition!)
• Simple DR is sufficient
• You need a readable copy (STANDBY mode)
• Compliance requires backups to remote location
• As a COMPLEMENT to Always On AG (belt + suspenders)

❌ DON'T USE LOG SHIPPING WHEN:
• You need automatic failover (use AG)
• You need zero data loss (use sync AG)
• You need sub-second RPO (use AG)
• You need read-only routing (use AG)
```

---

## ⚠️ Database Mirroring (Deprecated — Know It, Don't Use It)

```
╔══════════════════════════════════════════════════════════════════╗
║  DATABASE MIRRORING (Deprecated since SQL Server 2012)           ║
║                                                                  ║
║  Principal ◄──────────────────────────► Mirror                  ║
║  (Active)        Log records            (Standby)               ║
║                                                                  ║
║       ▲                                                          ║
║       │              ┌──────────┐                                ║
║       └──────────────│ WITNESS  │  (Optional — enables auto      ║
║                      │ (3rd srv)│   failover via quorum)         ║
║                      └──────────┘                                ║
║                                                                  ║
║  Modes:                                                          ║
║  • HIGH SAFETY (Synchronous) — zero data loss, with witness     ║
║  • HIGH PERFORMANCE (Async) — possible data loss, no witness    ║
║                                                                  ║
║  ⚠️ DEPRECATED — replaced by Always On AG                      ║
║  ⚠️ Limited to ONE database (AG can group multiple)             ║
║  ⚠️ No readable secondary (AG supports this)                   ║
║  ⚠️ Will be removed in future SQL Server version                ║
║                                                                  ║
║  💡 If you see this in production, plan migration to AG         ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🏗️ Designing HA/DR Architecture — Real-World Patterns

### Pattern 1: Standard Enterprise HA (Most Common)

```
┌──────────────────────────────────────────────────────────────────┐
│  PATTERN: 2-Node Synchronous AG + 1-Node Async DR               │
│  Cost: $$$                                                       │
│  RPO: 0 (local), minutes (DR)                                   │
│  RTO: ~30 seconds (local), minutes (DR)                         │
│                                                                  │
│  ┌─────────── DATA CENTER 1 ───────────┐   ┌─── DATA CENTER 2──┐│
│  │                                     │   │                    ││
│  │  ┌─────────┐    ┌─────────┐        │   │  ┌─────────┐      ││
│  │  │ Node 1  │◄──►│ Node 2  │        │   │  │ Node 3  │      ││
│  │  │ PRIMARY │sync│SECONDARY│        │   │  │SECONDARY│      ││
│  │  │ Auto-FO │    │ Auto-FO │        │   │  │Manual-FO│      ││
│  │  └─────────┘    └─────────┘        │   │  └─────────┘      ││
│  │       │              │              │   │       ▲            ││
│  │       └──────┬───────┘              │   │       │            ││
│  │              │                      │   │  Async commit      ││
│  │       ┌──────▼───────┐              │   │  (background)     ││
│  │       │   LISTENER   │              │   │                    ││
│  │       └──────────────┘              │   │                    ││
│  └─────────────────────────────────────┘   └────────────────────┘│
│                                                                  │
│  Normal: Apps → Listener → Node 1 (R/W), Node 2 (Read-Only)    │
│  Local Failure: Auto-failover to Node 2 (~30 sec)              │
│  DC1 Disaster: Manual failover to Node 3 (accept data loss)    │
└──────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Mission-Critical (Banking/Healthcare)

```
┌──────────────────────────────────────────────────────────────────┐
│  PATTERN: FCI + AG + Log Shipping (Belt + Suspenders + Helmet)  │
│  Cost: $$$$                                                      │
│  RPO: 0 everywhere                                               │
│  RTO: seconds                                                    │
│                                                                  │
│  ┌────── DATA CENTER 1 (Primary) ──────┐                        │
│  │                                     │                        │
│  │  ┌─────── FCI Cluster ──────────┐  │  ┌─── DATA CENTER 2 ──┐│
│  │  │ ┌────────┐  ┌────────┐      │  │  │                     ││
│  │  │ │Node 1A │  │Node 1B │      │  │  │  ┌────────┐        ││
│  │  │ │(Active)│  │(Passive│      │  │  │  │ Node 2 │        ││
│  │  │ │        │  │FCI     │      │──sync──│SECONDARY│        ││
│  │  │ │  SAN   │  │standby)│      │  │  │  │        │        ││
│  │  │ └────────┘  └────────┘      │  │  │  └────────┘        ││
│  │  └─────────────────────────────┘  │  │                     ││
│  │       This is the AG Primary       │  │  + Log Shipping     ││
│  │       (itself an FCI for           │  │    to tape vault    ││
│  │        hardware protection)        │  │                     ││
│  └─────────────────────────────────────┘  └─────────────────────┘│
│                                                                  │
│  Protection layers:                                              │
│  1. FCI protects against hardware failure (within DC1)          │
│  2. AG protects against everything (cross-DC)                   │
│  3. Log Shipping provides offline backup trail                  │
└──────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Budget-Friendly DR (SMB)

```
┌──────────────────────────────────────────────────────────────────┐
│  PATTERN: Log Shipping to a secondary server (Standard Edition) │
│  Cost: $                                                         │
│  RPO: 15-30 minutes                                             │
│  RTO: 30-60 minutes                                             │
│                                                                  │
│  ┌──── Office Server ─────┐    ┌──── Azure VM ─────────────┐   │
│  │  SQL Server Standard    │    │  SQL Server Standard      │   │
│  │  (Production)           │    │  (DR — Log Shipping)      │   │
│  │                         │    │                           │   │
│  │  Backup every 15 min ──►│    │  Restore every 15 min    │   │
│  │  to Azure Blob Storage  │    │  from Azure Blob Storage │   │
│  └─────────────────────────┘    └───────────────────────────┘   │
│                                                                  │
│  Cheapest DR option — works on Standard Edition!                │
│  + Azure VM can be deallocated when not needed (pay per use)    │
└──────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ WSFC Quorum — The Brain of the Cluster

```
╔══════════════════════════════════════════════════════════════════╗
║                    WSFC QUORUM MODELS                            ║
║                                                                  ║
║  Quorum = "How many votes needed to stay alive?"                ║
║  Rule: Majority of votes must be available (> 50%)              ║
║                                                                  ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  NODE MAJORITY (Odd number of nodes)                     │   ║
║  │  3 nodes → need 2 alive. 5 nodes → need 3 alive.        │   ║
║  │  Simple. No external witness needed.                     │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                                                                  ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  NODE + FILE SHARE WITNESS (Even number of nodes)        │   ║
║  │  2 nodes + 1 file share = 3 votes → need 2.             │   ║
║  │  ✅ Most common for 2-node clusters                     │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                                                                  ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  CLOUD WITNESS (Azure Blob Storage) — Recommended! ✅    │   ║
║  │  Uses Azure storage account as the witness.              │   ║
║  │  No need for a 3rd server or file share.                 │   ║
║  │  Works great for multi-subnet / cross-DC clusters.       │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🧪 Interview Questions — HA/DR

### Beginner Level

```
Q1: What is the difference between HA and DR?
A:  HA = keep running during small failures (same data center)
    DR = recover from catastrophic failures (different location)

Q2: What is an Always On Availability Group?
A:  A group of databases that replicate together to secondary 
    replicas. Supports sync/async commit, automatic failover,
    readable secondaries, and a listener for transparent 
    connection routing.

Q3: What is the difference between synchronous and asynchronous commit?
A:  Synchronous: Primary waits for secondary to confirm log 
    hardening → zero data loss, slightly slower
    Asynchronous: Primary commits immediately, sends log in 
    background → possible data loss, faster
```

### Advanced Level

```
Q4: You have a 2-node sync AG. Node 2 goes down. What happens?
A:  Node 1 (Primary) continues serving reads and writes.
    AG enters "Not Synchronized" state.
    Automatic failover is disabled (no sync partner).
    When Node 2 comes back, it catches up automatically.
    ⚠️ During this window, a Node 1 failure = data loss risk.

Q5: How would you perform a zero-downtime SQL Server upgrade?
A:  Rolling upgrade pattern:
    1. Patch Node 2 (secondary) first
    2. Manual failover: Node 2 becomes primary
    3. Patch Node 1 (now secondary)
    4. Optional: fail back to Node 1
    Result: zero downtime, zero data loss

Q6: Your AG secondary has a large redo queue (50GB). What do you do?
A:  Diagnose:
    • Check redo_rate — is the secondary I/O bottlenecked?
    • Check CPU pressure on secondary
    • Check if heavy read-only queries are blocking redo
    Fix:
    • Upgrade secondary storage (faster SSD)
    • Reduce read-only workload temporarily  
    • Increase max degree of parallelism for redo
    • Consider adding more secondary replicas to distribute reads

Q7: Can you use Always On AG with SQL Server Standard Edition?
A:  Yes, since SQL 2016 SP1 — "Basic Availability Groups":
    • Only 1 database per AG (not multiple)
    • Only 1 secondary replica
    • No readable secondary
    • No backup on secondary
    It's like Database Mirroring but future-proof.
```

---

## 🗺️ Decision Matrix — Choosing the Right HA/DR Solution

```
╔════════════════════════╦══════╦══════╦══════╦══════╦═══════════╗
║ Requirement            ║ AG   ║ FCI  ║ Log  ║ Mirr ║ Backup    ║
║                        ║      ║      ║ Ship ║ (dep)║ /Restore  ║
╠════════════════════════╬══════╬══════╬══════╬══════╬═══════════╣
║ Zero data loss         ║ ✅  ║ ✅  ║ ❌  ║ ✅  ║ ❌       ║
║ Automatic failover     ║ ✅  ║ ✅  ║ ❌  ║ ✅  ║ ❌       ║
║ Readable secondary     ║ ✅  ║ ❌  ║ ⚠️  ║ ❌  ║ ❌       ║
║ Multiple DB failover   ║ ✅  ║ ✅  ║ ❌  ║ ❌  ║ ❌       ║
║ Cross-datacenter       ║ ✅  ║ ⚠️  ║ ✅  ║ ✅  ║ ✅       ║
║ Standard Edition       ║ ⚠️  ║ ✅  ║ ✅  ║ ❌  ║ ✅       ║
║ No shared storage      ║ ✅  ║ ❌  ║ ✅  ║ ✅  ║ ✅       ║
║ Simplicity             ║ ⚠️  ║ ⚠️  ║ ✅  ║ ⚠️  ║ ✅       ║
║ Cost                   ║ $$$  ║ $$$  ║ $   ║ $$   ║ $         ║
╠════════════════════════╬══════╬══════╬══════╬══════╬═══════════╣
║ RECOMMENDATION (2026)  ║ 🥇  ║ 🥈  ║ 🥉  ║ ❌  ║ Always ✅║
║                        ║ Use  ║ When ║ Budg ║ Migr ║ (baseline)║
║                        ║ this ║ need ║ et   ║ away ║           ║
╚════════════════════════╩══════╩══════╩══════╩══════╩═══════════╝
```

---

## 🎯 Chapter Summary

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ✅ HA = survive local failures (seconds RTO)                  │
│     DR = survive catastrophic failures (minutes-hours RTO)     │
│                                                                │
│  ✅ Always On AG = THE recommended solution                    │
│     • Sync commit for HA (zero data loss)                      │
│     • Async commit for DR (possible data loss)                 │
│     • Readable secondaries, read-only routing                  │
│     • Listener for transparent connection management           │
│                                                                │
│  ✅ FCI = Instance-level HA with shared storage                │
│     • Can be combined with AG for maximum protection          │
│                                                                │
│  ✅ Log Shipping = Simple, cheap, reliable DR                  │
│     • Works on Standard Edition                                │
│     • Great complement to AG                                   │
│                                                                │
│  ✅ Database Mirroring = DEPRECATED → migrate to AG            │
│                                                                │
│  ✅ Every deployment MUST have backup/restore as baseline      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2C.7 — SQL Server Administration](./07-SQLServer-Admin.md) 🔴
