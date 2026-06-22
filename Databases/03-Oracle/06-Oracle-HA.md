# 🏰 Chapter 2B.6 — Oracle RAC, Data Guard & High Availability

> **Level:** 🔴 Advanced | 🔥 High Demand
> **Time to Master:** ~5-7 hours
> **Prerequisites:** Chapter 2B.1 (Oracle Architecture), Chapter 2B.5 (Performance Tuning)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why** enterprises pay millions for Oracle HA
- Explain **Real Application Clusters (RAC)** — multiple servers, one database
- Set up **Data Guard** — real-time disaster recovery across continents
- Design a **Maximum Availability Architecture (MAA)**
- Know the difference between **failover, switchover, and switchback**
- Handle **Active Data Guard** — read from your standby, not just wait

---

## 🧠 The Big Question

> *"What happens when your database server catches fire at 2 AM?"*

No, really. Data centers have fires, floods, power outages, and hardware failures. A single Oracle server, no matter how powerful, is a **single point of failure**.

```
╔══════════════════════════════════════════════════════════════════╗
║               THE COST OF DOWNTIME                               ║
║                                                                  ║
║   ┌─────────────────────────────────────────────┐               ║
║   │  Industry              │ Cost per Hour      │               ║
║   ├────────────────────────┼────────────────────┤               ║
║   │  Banking/Finance       │  $6.5 Million      │               ║
║   │  Telecom               │  $2.0 Million      │               ║
║   │  Manufacturing         │  $1.5 Million      │               ║
║   │  Healthcare            │  $636,000          │               ║
║   │  Retail/E-Commerce     │  $1.1 Million      │               ║
║   └────────────────────────┴────────────────────┘               ║
║                                                                  ║
║   When your database is down, your BUSINESS is down.            ║
║   Oracle's HA stack is INSURANCE against catastrophe.            ║
╚══════════════════════════════════════════════════════════════════╝
```

### Oracle HA Architecture — The Big Picture

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    ORACLE HIGH AVAILABILITY STACK                       ║
║                                                                          ║
║   ┌────────────────────────────────────────────────────────────┐        ║
║   │                    APPLICATION TIER                         │        ║
║   │    TAF / FCF / Application Continuity / FAN Events          │        ║
║   └────────────────────────────┬───────────────────────────────┘        ║
║                                │                                         ║
║   ┌────────────────────────────▼───────────────────────────────┐        ║
║   │                    RAC (Same Data Center)                   │        ║
║   │    ┌──────────┐    ┌──────────┐    ┌──────────┐           │        ║
║   │    │ Instance │    │ Instance │    │ Instance │           │        ║
║   │    │    1     │    │    2     │    │    3     │           │        ║
║   │    └────┬─────┘    └────┬─────┘    └────┬─────┘           │        ║
║   │         └───────────────┼───────────────┘                  │        ║
║   │                    ┌────▼────┐                              │        ║
║   │                    │ SHARED  │  ← ASM / Shared Storage     │        ║
║   │                    │ STORAGE │                              │        ║
║   │                    └─────────┘                              │        ║
║   └────────────────────────────────────────────────────────────┘        ║
║                                │                                         ║
║                         Redo Transport                                   ║
║                                │                                         ║
║   ┌────────────────────────────▼───────────────────────────────┐        ║
║   │            DATA GUARD (Remote Data Center)                  │        ║
║   │    ┌──────────────────────────────┐                         │        ║
║   │    │    STANDBY DATABASE          │                         │        ║
║   │    │    (Physical or Logical)     │                         │        ║
║   │    │                              │                         │        ║
║   │    │    Active Data Guard:        │                         │        ║
║   │    │    → Read-only queries ✓     │                         │        ║
║   │    │    → Real-time apply ✓       │                         │        ║
║   │    └──────────────────────────────┘                         │        ║
║   └────────────────────────────────────────────────────────────┘        ║
║                                                                          ║
║   RAC protects against:      SERVER failure (node goes down)            ║
║   Data Guard protects against: SITE failure (data center goes down)     ║
║   Together = Maximum Availability Architecture (MAA)                    ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

## 🔷 Section 1: Oracle RAC — Multiple Servers, One Database

### What is RAC?

**Real Application Clusters (RAC)** allows multiple Oracle instances (on different servers) to access the **same physical database** simultaneously. If one server dies, the others continue serving users.

```
╔══════════════════════════════════════════════════════════════════╗
║                  SINGLE INSTANCE vs RAC                          ║
║                                                                  ║
║   SINGLE INSTANCE:                RAC:                          ║
║                                                                  ║
║   ┌──────────────┐           ┌──────────┐  ┌──────────┐       ║
║   │  Instance    │           │Instance 1│  │Instance 2│       ║
║   │  (1 server)  │           │(Server A)│  │(Server B)│       ║
║   └──────┬───────┘           └────┬─────┘  └────┬─────┘       ║
║          │                        │              │              ║
║   ┌──────▼───────┐           ┌────▼──────────────▼────┐       ║
║   │   Database   │           │    SHARED DATABASE     │       ║
║   │   (1 copy)   │           │    (same datafiles)    │       ║
║   └──────────────┘           └────────────────────────┘       ║
║                                                                  ║
║   Server dies → 💀            Server A dies →                   ║
║   DATABASE DOWN!              Server B keeps running ✅          ║
║                               Users reconnect automatically     ║
╚══════════════════════════════════════════════════════════════════╝
```

### RAC Architecture Deep Dive

```
╔══════════════════════════════════════════════════════════════════════╗
║                      RAC COMPONENTS                                  ║
║                                                                      ║
║   ┌─────────────────────┐    ┌─────────────────────┐               ║
║   │     NODE 1          │    │     NODE 2          │               ║
║   │  ┌───────────────┐  │    │  ┌───────────────┐  │               ║
║   │  │ SGA           │  │    │  │ SGA           │  │               ║
║   │  │ ┌───────────┐ │  │    │  │ ┌───────────┐ │  │               ║
║   │  │ │Buffer Cache│ │  │    │  │ │Buffer Cache│ │  │               ║
║   │  │ │(local copy)│ │  │    │  │ │(local copy)│ │  │               ║
║   │  │ └───────────┘ │  │    │  │ └───────────┘ │  │               ║
║   │  │ ┌───────────┐ │  │    │  │ ┌───────────┐ │  │               ║
║   │  │ │ GCS / GES  │ │  │    │  │ │ GCS / GES  │ │  │               ║
║   │  │ │(Cache Fuse)│ │  │    │  │ │(Cache Fuse)│ │  │               ║
║   │  │ └───────────┘ │  │    │  │ └───────────┘ │  │               ║
║   │  └───────────────┘  │    │  └───────────────┘  │               ║
║   │                     │    │                     │               ║
║   │  REDO 1   UNDO 1   │    │  REDO 2   UNDO 2   │               ║
║   └──────────┬──────────┘    └──────────┬──────────┘               ║
║              │     PRIVATE                │                          ║
║              │  INTERCONNECT              │                          ║
║              │  (10GbE / InfiniBand)      │                          ║
║              │     ┌──────┐              │                          ║
║              └─────┤ FAST ├──────────────┘                          ║
║                    │ N/W  │  ← Cache Fusion transfers               ║
║                    └──────┘    blocks between nodes                  ║
║                       │                                              ║
║              ┌────────▼────────┐                                    ║
║              │  SHARED STORAGE │                                    ║
║              │  (ASM / SAN)    │                                    ║
║              │                 │                                    ║
║              │ ┌─────────────┐ │                                    ║
║              │ │ Datafiles   │ │ ← ALL nodes read/write same files  ║
║              │ │ Controlfile │ │                                    ║
║              │ │ SPFILE      │ │                                    ║
║              │ └─────────────┘ │                                    ║
║              └─────────────────┘                                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Key RAC Concepts

#### Cache Fusion — The Heart of RAC

```
Cache Fusion = sharing data blocks between nodes via the interconnect
             instead of writing to disk first.

SCENARIO: Node 2 needs a block that's modified in Node 1's buffer cache

WITHOUT Cache Fusion (old way):
   Node 1 → writes block to DISK → Node 2 reads from DISK
   Speed: SLOW (disk I/O)

WITH Cache Fusion:
   Node 1 → sends block via INTERCONNECT → Node 2 receives directly
   Speed: FAST (memory-to-memory, microseconds)

That's why RAC needs a FAST INTERCONNECT (InfiniBand preferred)!
```

#### Global Cache Service (GCS) & Global Enqueue Service (GES)

```
╔═════════════════════════════════════════════════════════════╗
║  GCS (Global Cache Service)                                 ║
║  → Manages data block transfers between nodes               ║
║  → Tracks which node has which block (and its state)        ║
║  → Uses "Global Resource Directory" (GRD)                   ║
║                                                             ║
║  GES (Global Enqueue Service)                               ║
║  → Manages locks across nodes                               ║
║  → Library cache locks, dictionary locks, row locks          ║
║  → Ensures no two nodes corrupt the same data               ║
╚═════════════════════════════════════════════════════════════╝
```

### RAC Wait Events to Watch

```
╔═══════════════════════════════════════════════════════════════════════╗
║  Wait Event                        │ Meaning                        ║
╠════════════════════════════════════╪═════════════════════════════════╣
║  gc buffer busy acquire            │ Block is in transit between    ║
║                                    │ nodes (interconnect busy)      ║
╠════════════════════════════════════╪═════════════════════════════════╣
║  gc buffer busy release            │ Local instance busy processing ║
║                                    │ the requested block            ║
╠════════════════════════════════════╪═════════════════════════════════╣
║  gc cr multi block request         │ Multi-block Cache Fusion read  ║
║                                    │ (full scans over interconnect) ║
╠════════════════════════════════════╪═════════════════════════════════╣
║  gc current block busy             │ Current (modified) block       ║
║                                    │ transfer taking too long       ║
╠════════════════════════════════════╪═════════════════════════════════╣
║  gc current grant busy             │ Wait for permission to modify  ║
║                                    │ a block held by another node   ║
╠════════════════════════════════════╪═════════════════════════════════╣
║  💡 FIX: If gc waits are high →                                    ║
║     1. Check interconnect bandwidth (InfiniBand > 10GbE)           ║
║     2. Reduce cross-node block contention (partition by node)      ║
║     3. Use service-based workload distribution                     ║
╚════════════════════════════════════╧═════════════════════════════════╝
```

### RAC Services — Smart Workload Distribution

```sql
-- Create a service that runs on Node 1 (preferred) with failover to Node 2
BEGIN
    DBMS_SERVICE.CREATE_SERVICE(
        service_name     => 'OLTP_SERVICE',
        network_name     => 'OLTP_SERVICE',
        failover_method  => 'BASIC',
        failover_type    => 'SELECT',
        failover_retries => 5,
        failover_delay   => 1
    );
END;
/

-- Using srvctl (Grid Infrastructure)
-- Create service
srvctl add service -db ORCL -service OLTP_SVC \
    -preferred ORCL1 -available ORCL2 \
    -failovermethod BASIC -failovertype SELECT

srvctl add service -db ORCL -service BATCH_SVC \
    -preferred ORCL2 -available ORCL1

-- Start/stop services
srvctl start service -db ORCL -service OLTP_SVC
srvctl stop service -db ORCL -service OLTP_SVC

-- Check service status
srvctl status service -db ORCL
```

```
╔══════════════════════════════════════════════════════════════════╗
║            RAC SERVICE-BASED WORKLOAD DISTRIBUTION               ║
║                                                                  ║
║   ┌──────────────────┐     ┌──────────────────┐                ║
║   │  OLTP Users      │     │  Batch/Report     │                ║
║   │  (Fast queries)  │     │  (Heavy queries)  │                ║
║   └────────┬─────────┘     └────────┬──────────┘                ║
║            │                        │                            ║
║            ▼                        ▼                            ║
║   ┌────────────────┐     ┌─────────────────┐                    ║
║   │  OLTP_SERVICE  │     │  BATCH_SERVICE  │                    ║
║   │  → Node 1 (P)  │     │  → Node 2 (P)  │                    ║
║   │  → Node 2 (A)  │     │  → Node 1 (A)  │                    ║
║   └────────────────┘     └─────────────────┘                    ║
║                                                                  ║
║   P = Preferred node (normal traffic goes here)                  ║
║   A = Available node (failover target)                           ║
║                                                                  ║
║   Result: Workloads don't compete for the same resources!       ║
╚══════════════════════════════════════════════════════════════════╝
```

### Application Continuity — Zero Downtime Failover

```
╔══════════════════════════════════════════════════════════════════╗
║              APPLICATION CONTINUITY (AC)                         ║
║                                                                  ║
║   Traditional Failover:                                          ║
║     Node 1 dies → User gets ERROR → Must retry → Bad UX 😤     ║
║                                                                  ║
║   With Application Continuity:                                   ║
║     Node 1 dies → Oracle REPLAYS the in-flight transaction      ║
║     on Node 2 → User never notices → Seamless! 🎯              ║
║                                                                  ║
║   How it works:                                                  ║
║   1. Oracle records all calls in a "request" queue               ║
║   2. If node fails mid-transaction, the surviving node           ║
║      replays the recorded calls                                  ║
║   3. Non-idempotent operations are detected and handled          ║
║   4. User sees ZERO errors                                       ║
║                                                                  ║
║   ⚠️ Requires: Thin JDBC driver + AC-enabled service            ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔷 Section 2: Oracle Data Guard — Disaster Recovery

### What is Data Guard?

Data Guard maintains one or more **standby databases** that are synchronized copies of the **primary database**. If the primary goes down (fire, earthquake, ransomware), you **switch to the standby** in seconds or minutes.

```
╔══════════════════════════════════════════════════════════════════════╗
║                   DATA GUARD ARCHITECTURE                            ║
║                                                                      ║
║   PRIMARY SITE (Mumbai)            STANDBY SITE (Hyderabad)         ║
║                                                                      ║
║   ┌───────────────────┐           ┌───────────────────┐             ║
║   │  PRIMARY DATABASE │           │ STANDBY DATABASE  │             ║
║   │                   │   Redo    │                   │             ║
║   │  Reads ✅         │──────────►│  Applies redo     │             ║
║   │  Writes ✅        │ Transport │  continuously     │             ║
║   │                   │           │                   │             ║
║   │  Redo Logs ──────────────────►│  Redo applied     │             ║
║   │                   │           │  in real-time     │             ║
║   └───────────────────┘           └───────────────────┘             ║
║                                                                      ║
║   Primary goes DOWN →            Standby becomes PRIMARY            ║
║   (Failover/Switchover)          → Users connect here now           ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Physical vs Logical Standby

```
╔════════════════════════════════════════════════════════════════════╗
║                │  Physical Standby       │  Logical Standby       ║
╠════════════════╪═════════════════════════╪════════════════════════╣
║  Sync Method   │  Block-for-block copy   │  SQL replay            ║
║                │  (redo apply)           │  (SQL apply)           ║
╠════════════════╪═════════════════════════╪════════════════════════╣
║  Identical     │  YES (exact binary      │  NO (logically same,   ║
║  to Primary?   │  copy)                  │  physically different) ║
╠════════════════╪═════════════════════════╪════════════════════════╣
║  Can add       │  NO (read-only, or      │  YES (can add indexes, ║
║  extra objects?│  read-write with ADG)   │  tables, etc.)         ║
╠════════════════╪═════════════════════════╪════════════════════════╣
║  Failover      │  Very fast (seconds)    │  Slower (minutes)      ║
║  Speed         │                         │                        ║
╠════════════════╪═════════════════════════╪════════════════════════╣
║  Use Case      │  DR, read offloading,   │  Reporting, upgrades,  ║
║                │  testing                │  different schema needs║
╠════════════════╪═════════════════════════╪════════════════════════╣
║  ⭐ Most       │  ✅ 95% of setups       │  5% — special cases    ║
║  Common?       │  use physical standby   │                        ║
╚════════════════╧═════════════════════════╧════════════════════════╝
```

### Data Guard Protection Modes

```
╔══════════════════════════════════════════════════════════════════════════╗
║  Mode              │ Behavior                    │ Data Loss │ Perf   ║
╠════════════════════╪═════════════════════════════╪═══════════╪════════╣
║  MAXIMUM           │ Primary WAITS for standby   │ ZERO      │ Slower ║
║  PROTECTION        │ to confirm redo received.   │           │        ║
║                    │ Primary STOPS if standby    │ (safest)  │        ║
║                    │ is unreachable.             │           │        ║
╠════════════════════╪═════════════════════════════╪═══════════╪════════╣
║  MAXIMUM           │ Primary WAITS for standby   │ ZERO      │ Medium ║
║  AVAILABILITY      │ to confirm redo received.   │ (normally)│        ║
║  (Recommended ⭐)  │ But if standby is down,     │           │        ║
║                    │ primary continues alone.    │           │        ║
╠════════════════════╪═════════════════════════════╪═══════════╪════════╣
║  MAXIMUM           │ Primary sends redo ASYNC.   │ Possible  │ Fastest║
║  PERFORMANCE       │ Doesn't wait for standby    │ (seconds) │        ║
║  (Default)         │ confirmation.               │           │        ║
╚════════════════════╧═════════════════════════════╧═══════════╧════════╝

💡 Most enterprises use MAXIMUM AVAILABILITY:
   → Zero data loss during normal operation
   → Primary stays available even if standby is temporarily down
```

### Setting Up Data Guard — Key Steps

```sql
-- Step 1: Configure Primary Database
-- Enable force logging
ALTER DATABASE FORCE LOGGING;

-- Set up standby redo logs (1 more group than online redo logs)
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 SIZE 200M;

-- Configure Data Guard parameters
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(PROD,STBY)';
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1='LOCATION=USE_DB_RECOVERY_FILE_DEST
    VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=PROD';
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=STBY ASYNC
    VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=STBY';
ALTER SYSTEM SET FAL_SERVER='STBY';
ALTER SYSTEM SET DB_FILE_NAME_CONVERT='/stby/','/prod/' SCOPE=SPFILE;
ALTER SYSTEM SET LOG_FILE_NAME_CONVERT='/stby/','/prod/' SCOPE=SPFILE;

-- Step 2: Create Standby (using RMAN duplicate)
-- On the standby server:
RMAN TARGET sys/password@PROD AUXILIARY sys/password@STBY
DUPLICATE TARGET DATABASE
    FOR STANDBY
    FROM ACTIVE DATABASE
    DORECOVER
    NOFILENAMECHECK;

-- Step 3: Start Redo Apply on Standby
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
    USING CURRENT LOGFILE DISCONNECT;

-- Step 4: Verify Data Guard status
SELECT DATABASE_ROLE, PROTECTION_MODE, PROTECTION_LEVEL
FROM V$DATABASE;

-- Check standby apply lag
SELECT NAME, VALUE, DATUM_TIME
FROM V$DATAGUARD_STATS
WHERE NAME IN ('transport lag', 'apply lag');
```

### Switchover vs Failover

```
╔══════════════════════════════════════════════════════════════════════╗
║              SWITCHOVER                     FAILOVER                ║
║                                                                      ║
║   ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐         ║
║   │PRIMARY │     │STANDBY │     │PRIMARY │     │STANDBY │         ║
║   │   A    │     │   B    │     │   A    │     │   B    │         ║
║   └───┬────┘     └───┬────┘     └───┬────┘     └───┬────┘         ║
║       │              │              │              │                ║
║       ▼              ▼              ▼              ▼                ║
║   ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐         ║
║   │STANDBY │     │PRIMARY │     │  DEAD  │     │PRIMARY │         ║
║   │   A    │     │   B    │     │  💀 A  │     │   B    │         ║
║   └────────┘     └────────┘     └────────┘     └────────┘         ║
║                                                                      ║
║   PLANNED             PLANNED     UNPLANNED        UNPLANNED       ║
║   No data loss ✅     No data loss Possible data   Possible data   ║
║   Reversible ✅       Reversible   loss ⚠️         loss ⚠️         ║
║                                   NOT reversible  NOT reversible   ║
║                                   (use REINSTATE) (use REINSTATE)  ║
╚══════════════════════════════════════════════════════════════════════╝
```

```sql
-- SWITCHOVER (planned, graceful)
-- On Primary:
ALTER DATABASE COMMIT TO SWITCHOVER TO STANDBY WITH SESSION SHUTDOWN;

-- On Standby (becomes new Primary):
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
ALTER DATABASE OPEN;

-- FAILOVER (emergency, primary is dead)
-- On Standby:
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;  -- Apply remaining redo
ALTER DATABASE ACTIVATE STANDBY DATABASE;
ALTER DATABASE OPEN;

-- Using Data Guard Broker (RECOMMENDED — much easier!)
-- Switchover:
DGMGRL> SWITCHOVER TO 'STBY';

-- Failover:
DGMGRL> FAILOVER TO 'STBY';

-- Fast-Start Failover (AUTOMATIC failover!)
DGMGRL> ENABLE FAST_START FAILOVER;
-- Observer process monitors primary → auto-failover if primary goes down
```

### Data Guard Broker — Simplifying Management

```sql
-- Data Guard Broker centralizes management of the entire DG configuration

-- Create broker configuration
DGMGRL> CREATE CONFIGURATION 'DG_CONFIG' AS
         PRIMARY DATABASE IS 'PROD'
         CONNECT IDENTIFIER IS 'PROD';

DGMGRL> ADD DATABASE 'STBY' AS
         CONNECT IDENTIFIER IS 'STBY'
         MAINTAINED AS PHYSICAL;

DGMGRL> ENABLE CONFIGURATION;

-- Show configuration status
DGMGRL> SHOW CONFIGURATION;

-- Expected output:
-- Configuration - DG_CONFIG
--   Protection Mode: MaxAvailability
--   Members:
--     PROD - Primary database
--     STBY - Physical standby database
--   Fast-Start Failover: Disabled
-- Configuration Status:
--   SUCCESS

-- Show database details
DGMGRL> SHOW DATABASE 'STBY';

-- Show apply lag
DGMGRL> SHOW DATABASE 'STBY' 'StatusReport';
```

---

## 🔷 Section 3: Active Data Guard — Put Your Standby to Work

### Why Active Data Guard?

Traditional standby databases sit **idle** — they're just waiting for a disaster. **Active Data Guard** lets you **read from the standby** while it continues applying redo. Your standby becomes a **productive resource**.

```
╔══════════════════════════════════════════════════════════════════╗
║     TRADITIONAL STANDBY           ACTIVE DATA GUARD             ║
║                                                                  ║
║   ┌───────────┐                 ┌───────────┐                   ║
║   │  STANDBY  │                 │  STANDBY  │                   ║
║   │           │                 │           │                   ║
║   │ ❌ Reads   │                 │ ✅ Reads   │                   ║
║   │ ❌ Reports │                 │ ✅ Reports │                   ║
║   │ ❌ Backups │                 │ ✅ Backups │                   ║
║   │           │                 │ ✅ Apply   │                   ║
║   │ 💤 Idle   │                 │           │                   ║
║   │ (wasting  │                 │ 💪 Working │                   ║
║   │  $$$$)    │                 │ (earning   │                   ║
║   └───────────┘                 │  its keep) │                   ║
║                                 └───────────┘                   ║
║                                                                  ║
║   You pay for standby hardware → Make it WORK for you!          ║
╚══════════════════════════════════════════════════════════════════╝
```

### Active Data Guard Features

```sql
-- Open standby for READ-ONLY while applying redo
ALTER DATABASE OPEN READ ONLY;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
    USING CURRENT LOGFILE DISCONNECT;
-- Now: redo apply is running AND queries are being served!

-- Route reporting queries to standby (via service)
srvctl add service -db STBY -service REPORT_SVC -role PHYSICAL_STANDBY

-- On Primary — create service that auto-starts on standby
BEGIN
    DBMS_SERVICE.CREATE_SERVICE(
        service_name => 'REPORTING',
        network_name => 'REPORTING'
    );
END;
/
```

### Active Data Guard — Far Sync Instance

```
╔══════════════════════════════════════════════════════════════════════╗
║              FAR SYNC — Zero Data Loss Over Any Distance            ║
║                                                                      ║
║   Problem: SYNC redo transport to distant standby = high latency    ║
║                                                                      ║
║   Solution: Far Sync instance acts as a RELAY                       ║
║                                                                      ║
║   Mumbai              Pune (nearby)           US-East (far)         ║
║   ┌─────────┐   SYNC  ┌──────────┐   ASYNC  ┌─────────┐          ║
║   │ PRIMARY │────────►│ FAR SYNC │─────────►│ STANDBY │          ║
║   │         │  (fast) │(no data, │  (slow   │         │          ║
║   │         │         │ relay    │   but ok) │         │          ║
║   │         │         │ only)    │           │         │          ║
║   └─────────┘         └──────────┘           └─────────┘          ║
║                                                                      ║
║   Primary → Far Sync: SYNCHRONOUS (zero data loss, low latency)    ║
║   Far Sync → Standby: ASYNCHRONOUS (distance doesn't matter)       ║
║   Result: ZERO DATA LOSS with a distant standby! 🎯               ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔷 Section 4: Maximum Availability Architecture (MAA)

### What is MAA?

**MAA** is Oracle's **best-practices blueprint** for achieving the highest levels of availability. It combines multiple technologies into a unified architecture.

```
╔══════════════════════════════════════════════════════════════════════════╗
║              MAXIMUM AVAILABILITY ARCHITECTURE (MAA)                    ║
║                                                                          ║
║   ┌──────────────────── DATA CENTER 1 ──────────────────────┐           ║
║   │                                                          │           ║
║   │   ┌──────────────── RAC CLUSTER ─────────────────┐      │           ║
║   │   │                                               │      │           ║
║   │   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │      │           ║
║   │   │  │  Node 1  │  │  Node 2  │  │  Node 3  │  │      │           ║
║   │   │  │ Instance │  │ Instance │  │ Instance │  │      │           ║
║   │   │  └──────────┘  └──────────┘  └──────────┘  │      │           ║
║   │   │           ↕    Cache Fusion    ↕             │      │           ║
║   │   │                                               │      │           ║
║   │   │  ┌─────────────────────────────────────────┐ │      │           ║
║   │   │  │        ASM (Oracle ASM)                  │ │      │           ║
║   │   │  │  Normal / High Redundancy Disk Groups   │ │      │           ║
║   │   │  └─────────────────────────────────────────┘ │      │           ║
║   │   └───────────────────────────────────────────────┘      │           ║
║   │                          │                                │           ║
║   └──────────────────────────┼────────────────────────────────┘           ║
║                              │ Redo Transport                             ║
║                              │ (SYNC or ASYNC)                            ║
║                              ▼                                            ║
║   ┌──────────────────── DATA CENTER 2 ──────────────────────┐           ║
║   │                                                          │           ║
║   │   ┌──────────────── RAC CLUSTER ─────────────────┐      │           ║
║   │   │                                               │      │           ║
║   │   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │      │           ║
║   │   │  │  Node 4  │  │  Node 5  │  │  Node 6  │  │      │           ║
║   │   │  │ Standby  │  │ Standby  │  │ Standby  │  │      │           ║
║   │   │  └──────────┘  └──────────┘  └──────────┘  │      │           ║
║   │   │                                               │      │           ║
║   │   │  Active Data Guard (read-only queries OK)    │      │           ║
║   │   └───────────────────────────────────────────────┘      │           ║
║   │                                                          │           ║
║   └──────────────────────────────────────────────────────────┘           ║
║                                                                          ║
║   Protection Against:                                                    ║
║   ├── Node failure        → RAC handles (automatic)                     ║
║   ├── Storage failure     → ASM mirroring handles                       ║
║   ├── Data center failure → Data Guard switchover/failover              ║
║   ├── Human error         → Flashback Database (undo changes)           ║
║   ├── Data corruption     → Block-level validation + repair             ║
║   └── Planned maintenance → Rolling upgrades (RAC + DG)                 ║
║                                                                          ║
║   Target: 99.999% uptime (5 minutes of downtime per YEAR)              ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### MAA Tiers

```
╔══════════════════════════════════════════════════════════════════════════╗
║  Tier     │ Components                    │ RTO        │ RPO           ║
╠═══════════╪═══════════════════════════════╪════════════╪═══════════════╣
║  Bronze   │ Single instance + RMAN Backup │ Hours      │ Last backup   ║
║           │                               │            │ (hours)       ║
╠═══════════╪═══════════════════════════════╪════════════╪═══════════════╣
║  Silver   │ RAC + Local Data Guard        │ Minutes    │ Near-zero     ║
║           │                               │            │               ║
╠═══════════╪═══════════════════════════════╪════════════╪═══════════════╣
║  Gold     │ RAC + Active Data Guard       │ Seconds    │ Zero          ║
║           │ (Sync transport)              │            │               ║
╠═══════════╪═══════════════════════════════╪════════════╪═══════════════╣
║  Platinum │ RAC + ADG + Far Sync +        │ Zero       │ Zero          ║
║           │ Fast-Start Failover +         │ (automatic │               ║
║           │ Application Continuity        │  failover) │               ║
╚═══════════╧═══════════════════════════════╧════════════╧═══════════════╝

RTO = Recovery Time Objective (how fast you recover)
RPO = Recovery Point Objective (how much data you can afford to lose)
```

---

## 🔷 Section 5: Flashback Technologies — Oracle's Time Machine

### Flashback Database

```sql
-- Undo an entire database to a point in time (faster than restore!)

-- Enable Flashback Database
ALTER SYSTEM SET DB_FLASHBACK_RETENTION_TARGET = 4320;  -- 3 days (minutes)
ALTER DATABASE FLASHBACK ON;

-- Flash back the database to 2 hours ago
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
FLASHBACK DATABASE TO TIMESTAMP (SYSTIMESTAMP - INTERVAL '2' HOUR);
ALTER DATABASE OPEN RESETLOGS;

-- Or flash back to a specific SCN
FLASHBACK DATABASE TO SCN 123456789;
```

### Other Flashback Features

```sql
-- Flashback Query — See data as it was in the past
SELECT employee_name, salary
FROM employees AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR)
WHERE employee_id = 100;

-- Flashback Table — Undo a table to a previous state
FLASHBACK TABLE employees TO TIMESTAMP
    (SYSTIMESTAMP - INTERVAL '30' MINUTE);

-- Flashback Drop — Recover a dropped table
FLASHBACK TABLE employees TO BEFORE DROP;

-- Flashback Transaction — Undo a specific transaction
-- (Requires supplemental logging)
```

---

## 🧪 Interview-Ready Explanations

### "Explain Oracle RAC in 30 seconds"

> *"Oracle RAC allows multiple server instances to share a single database. All nodes can read and write simultaneously. If one node fails, the others continue operating — users are automatically reconnected via TAF or Application Continuity. The nodes communicate through a high-speed interconnect using Cache Fusion, which transfers data blocks directly between memory pools without touching disk."*

### "Explain Data Guard in 30 seconds"

> *"Data Guard maintains a real-time copy of the primary database at a remote site by shipping redo logs. In Maximum Availability mode, it guarantees zero data loss. If the primary site has a disaster, you can failover to the standby in seconds. Active Data Guard additionally lets you run read-only queries on the standby, so it's not sitting idle."*

### Quick-Fire Interview Questions

| Question | Answer |
|----------|--------|
| "RAC vs Data Guard?" | RAC protects against node failure (same site). Data Guard protects against site failure (different sites). Use both together for MAA. |
| "Switchover vs Failover?" | Switchover = planned, graceful, reversible, zero data loss. Failover = emergency, when primary is dead, possible data loss. |
| "What is Cache Fusion?" | RAC mechanism to transfer data blocks between nodes via the interconnect (memory-to-memory), avoiding slow disk I/O. |
| "Protection modes?" | Maximum Protection (zero loss, primary stops if standby unreachable), Maximum Availability (zero loss, primary continues if standby down), Maximum Performance (async, possible loss). |
| "What is Application Continuity?" | Oracle replays in-flight transactions on a surviving RAC node so the end user sees zero errors during failover. |
| "Physical vs Logical standby?" | Physical = block-for-block copy (redo apply). Logical = SQL replay (can have additional objects). Physical is 95% of deployments. |
| "What is Far Sync?" | A lightweight instance that relays redo synchronously from primary, then asynchronously to a distant standby. Achieves zero data loss over any distance. |

---

## ⚠️ Common Misconceptions

```
╔════════════════════════════════════════════════════════════════════════╗
║ MYTH                            │ REALITY                            ║
╠═════════════════════════════════╪════════════════════════════════════╣
║ "RAC doubles performance"       │ RAC provides HA, not linear       ║
║                                 │ scaling. Interconnect overhead    ║
║                                 │ means ~1.7x for 2 nodes, not 2x  ║
╠═════════════════════════════════╪════════════════════════════════════╣
║ "Data Guard = backup"           │ DG is NOT a backup. It protects   ║
║                                 │ against site failure but not      ║
║                                 │ user error (DELETE replicates!)   ║
╠═════════════════════════════════╪════════════════════════════════════╣
║ "Standby is wasted hardware"    │ Active Data Guard runs queries,   ║
║                                 │ backups, and reports on standby   ║
╠═════════════════════════════════╪════════════════════════════════════╣
║ "Failover is instant"           │ Fast-Start Failover: ~10-30 sec   ║
║                                 │ Manual failover: 1-5 minutes      ║
╠═════════════════════════════════╪════════════════════════════════════╣
║ "You need RAC + DG"             │ Small shops: DG alone is often    ║
║                                 │ enough. RAC adds HA but also      ║
║                                 │ complexity and cost               ║
╚═════════════════════════════════╧════════════════════════════════════╝
```

---

## 🔑 Key Takeaways

```
✅ RAC = Multiple instances, one database — protects against NODE failure
✅ Data Guard = Primary + Standby(s) — protects against SITE failure
✅ Cache Fusion = Memory-to-memory block transfer between RAC nodes
✅ Active Data Guard = Read queries from standby while it applies redo
✅ Protection Modes: Max Protection > Max Availability > Max Performance
✅ Switchover = planned (zero loss) | Failover = emergency (possible loss)
✅ Application Continuity = Zero user-visible errors during RAC failover
✅ Far Sync = Zero data loss over any distance via relay instance
✅ MAA = RAC + Data Guard + ASM + Flashback = 99.999% uptime
✅ Flashback Database = Oracle's time machine (faster than full restore)
✅ Always use Data Guard Broker (DGMGRL) for simplified management
```

---

## 🔗 What's Next?

**Chapter 2B.7 → [Oracle Administration & Maintenance](./07-Oracle-Admin.md)**
Where we learn to manage **tablespaces, users, backups, and patching** — the daily life of an Oracle DBA.

---

> *"High availability is not a feature you add later — it's an architecture you design from day one."* — Oracle MAA Best Practices
