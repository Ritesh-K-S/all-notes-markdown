# 🏗️ Chapter 2B.8 — Oracle Multitenant & Pluggable Databases

> **Level:** 🔴 Advanced
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 2B.1 (Oracle Architecture), Chapter 2B.7 (Administration)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the **Multitenant Architecture** (CDB/PDB) — Oracle's biggest architectural shift
- Create, clone, plug, and unplug **Pluggable Databases** like a pro
- Manage **resource isolation** between PDBs (so one rogue app doesn't kill the others)
- Perform **PDB-level backups, recovery, and flashback**
- Know why this architecture saves enterprises **millions** in hardware and licensing

---

## 🧠 The Big Question

> *"We have 50 Oracle databases in our company. Each needs its own server, its own memory, its own DBA time, its own patch cycle. Is there a better way?"*

**YES.** Oracle Multitenant (introduced in 12c, matured in 19c/21c) is the answer. Instead of 50 separate databases on 50 servers, you run **one Container Database (CDB)** with **50 Pluggable Databases (PDBs)** inside it.

```
╔══════════════════════════════════════════════════════════════════════╗
║            THE OLD WAY (Pre-12c)         THE MULTITENANT WAY       ║
║                                                                      ║
║   ┌──────┐ ┌──────┐ ┌──────┐          ┌──────────────────────┐    ║
║   │ DB 1 │ │ DB 2 │ │ DB 3 │          │ CONTAINER DB (CDB)  │    ║
║   │      │ │      │ │      │          │                      │    ║
║   │ SGA  │ │ SGA  │ │ SGA  │          │ Shared SGA           │    ║
║   │ 16GB │ │ 16GB │ │ 16GB │          │ 32GB (shared!)       │    ║
║   │      │ │      │ │      │          │                      │    ║
║   │ BG   │ │ BG   │ │ BG   │          │ ┌─────┐┌─────┐┌────┐│    ║
║   │ Proc │ │ Proc │ │ Proc │          │ │PDB 1││PDB 2││PDB3││    ║
║   └──┬───┘ └──┬───┘ └──┬───┘          │ │ HR  ││ FIN ││ CRM││    ║
║      │        │        │              │ └─────┘└─────┘└────┘│    ║
║   ┌──▼───┐ ┌──▼───┐ ┌──▼───┐          └──────────┬───────────┘    ║
║   │Server│ │Server│ │Server│                     │                  ║
║   │  1   │ │  2   │ │  3   │          ┌──────────▼───────────┐    ║
║   └──────┘ └──────┘ └──────┘          │   ONE Server         │    ║
║                                        │   (or RAC cluster)   │    ║
║   3 servers × $$$                      └──────────────────────┘    ║
║   3 × patching effort                                              ║
║   3 × backup config                  1 server, 1 patch cycle       ║
║   3 × memory overhead                1 set of background procs     ║
║                                       MASSIVE savings! 💰          ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🏛️ Section 1: CDB/PDB Architecture — The Full Picture

### The Three Containers

```
╔══════════════════════════════════════════════════════════════════════╗
║                  MULTITENANT ARCHITECTURE                            ║
║                                                                      ║
║   ┌──────────────────────────────────────────────────────────┐      ║
║   │                  CDB (Container Database)                 │      ║
║   │                                                           │      ║
║   │   ┌────────────────────────────────┐                     │      ║
║   │   │    CDB$ROOT (Root Container)   │                     │      ║
║   │   │                                │                     │      ║
║   │   │  • Oracle-supplied metadata    │                     │      ║
║   │   │  • Data dictionary (master)    │                     │      ║
║   │   │  • Common users (C##USER)      │                     │      ║
║   │   │  • Common roles                │                     │      ║
║   │   │                                │                     │      ║
║   │   │  ⚠️ NEVER put app data here!   │                     │      ║
║   │   └────────────────────────────────┘                     │      ║
║   │                                                           │      ║
║   │   ┌────────────────────────────────┐                     │      ║
║   │   │    PDB$SEED (Seed Container)   │                     │      ║
║   │   │                                │                     │      ║
║   │   │  • Template for new PDBs       │                     │      ║
║   │   │  • READ-ONLY (can't modify)    │                     │      ║
║   │   │  • Clone this to create PDBs   │                     │      ║
║   │   └────────────────────────────────┘                     │      ║
║   │                                                           │      ║
║   │   ┌──────────┐  ┌──────────┐  ┌──────────┐             │      ║
║   │   │  PDB_HR  │  │ PDB_FIN  │  │ PDB_CRM  │  ...        │      ║
║   │   │          │  │          │  │          │             │      ║
║   │   │ HR App   │  │ Finance  │  │ CRM App  │             │      ║
║   │   │ Tables   │  │ Tables   │  │ Tables   │             │      ║
║   │   │ Users    │  │ Users    │  │ Users    │             │      ║
║   │   │ Indexes  │  │ Indexes  │  │ Indexes  │             │      ║
║   │   └──────────┘  └──────────┘  └──────────┘             │      ║
║   │                                                           │      ║
║   │   Each PDB looks like a STANDALONE database to the app   │      ║
║   └──────────────────────────────────────────────────────────┘      ║
║                                                                      ║
║   KEY INSIGHT: The application connects to a PDB, not the CDB.     ║
║   From the app's perspective, it's just a normal Oracle database.   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### What's Shared vs What's Separate

```
╔════════════════════════════════════════════════════════════════════╗
║                    │  SHARED (CDB-level)   │  SEPARATE (per PDB) ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Memory (SGA/PGA)  │  ✅ Shared pool       │  Can set per-PDB    ║
║                    │     across all PDBs   │  memory limits      ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Background        │  ✅ One set of LGWR,  │  —                  ║
║  Processes         │     DBWR, PMON, etc.  │                     ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Redo Logs         │  ✅ Shared redo logs  │  —                  ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Undo Tablespace   │  ✅ Shared (default)  │  Per-PDB undo       ║
║                    │     or per-PDB       │  (Local Undo Mode)  ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Data Dictionary   │  Master copy in ROOT │  Local copy + link  ║
║                    │                       │  to ROOT             ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Datafiles         │  —                    │  ✅ Own datafiles    ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Temp Tablespace   │  Shared or per-PDB   │  ✅ Can be per-PDB   ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Users/Schemas     │  Common users (C##)  │  ✅ Local users       ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  Application Data  │  —                    │  ✅ Fully isolated   ║
╠════════════════════╪═══════════════════════╪═════════════════════╣
║  SPFILE            │  ✅ One SPFILE         │  Per-PDB parameters ║
║                    │                       │  stored in SPFILE   ║
╚════════════════════╧═══════════════════════╧═════════════════════╝
```

---

## 🔧 Section 2: Creating and Managing PDBs

### Creating a PDB

```sql
-- ═══════════════════════════════════════
-- METHOD 1: Create from PDB$SEED (most common ⭐)
-- ═══════════════════════════════════════

CREATE PLUGGABLE DATABASE pdb_hr
    ADMIN USER pdb_admin IDENTIFIED BY "SecureP@ss123!"
    ROLES = (DBA)
    DEFAULT TABLESPACE users
        DATAFILE '/u01/oradata/CDB1/pdb_hr/users01.dbf' SIZE 5G AUTOEXTEND ON
    FILE_NAME_CONVERT = ('/pdbseed/', '/pdb_hr/');

-- Open the PDB
ALTER PLUGGABLE DATABASE pdb_hr OPEN;

-- Save the state so PDB opens automatically on CDB restart
ALTER PLUGGABLE DATABASE pdb_hr SAVE STATE;

-- ═══════════════════════════════════════
-- METHOD 2: Clone an existing PDB (hot clone — source stays open!)
-- ═══════════════════════════════════════

CREATE PLUGGABLE DATABASE pdb_hr_dev FROM pdb_hr
    FILE_NAME_CONVERT = ('/pdb_hr/', '/pdb_hr_dev/')
    NO DATA;          -- Structure only, no data (for dev environments)
-- Without NO DATA → full clone with all data

-- Full clone (with data)
CREATE PLUGGABLE DATABASE pdb_hr_qa FROM pdb_hr
    FILE_NAME_CONVERT = ('/pdb_hr/', '/pdb_hr_qa/');

-- ═══════════════════════════════════════
-- METHOD 3: Clone from a remote CDB (over DB link!)
-- ═══════════════════════════════════════

-- Create a DB link to the remote CDB
CREATE DATABASE LINK remote_cdb
    CONNECT TO c##clone_user IDENTIFIED BY "password"
    USING 'REMOTE_CDB';

-- Clone the remote PDB
CREATE PLUGGABLE DATABASE pdb_hr_clone FROM pdb_hr@remote_cdb
    FILE_NAME_CONVERT = ('/remote_path/', '/local_path/');

-- ═══════════════════════════════════════
-- METHOD 4: Relocate a PDB (MOVE from one CDB to another — 12.2+)
-- ═══════════════════════════════════════

CREATE PLUGGABLE DATABASE pdb_hr FROM pdb_hr@remote_cdb RELOCATE
    FILE_NAME_CONVERT = ('/remote_path/', '/local_path/');
-- The PDB is MOVED — it's removed from the source CDB!
```

### PDB Lifecycle — Unplug and Plug

This is the **killer feature** of Multitenant — you can unplug a database like a USB drive and plug it into another CDB.

```
╔══════════════════════════════════════════════════════════════════════╗
║                  UNPLUG AND PLUG WORKFLOW                            ║
║                                                                      ║
║   CDB 1 (Production)                CDB 2 (Test/Dev)               ║
║                                                                      ║
║   ┌─────────────────┐               ┌─────────────────┐            ║
║   │  ┌───────────┐  │               │                 │            ║
║   │  │  PDB_HR   │──┼── UNPLUG ──►  │  ┌───────────┐  │            ║
║   │  │           │  │   (XML +      │  │  PDB_HR   │  │            ║
║   │  │  Tables   │  │   datafiles)  │  │  (plugged) │  │            ║
║   │  │  Data     │  │               │  │           │  │            ║
║   │  └───────────┘  │               │  └───────────┘  │            ║
║   │                 │               │                 │            ║
║   └─────────────────┘               └─────────────────┘            ║
║                                                                      ║
║   Like unplugging a USB drive and plugging it into another laptop!  ║
╚══════════════════════════════════════════════════════════════════════╝
```

```sql
-- ═══════════════════════════════════════
-- UNPLUG a PDB (exports metadata to XML)
-- ═══════════════════════════════════════

-- Close the PDB first
ALTER PLUGGABLE DATABASE pdb_hr CLOSE IMMEDIATE;

-- Unplug (creates an XML manifest file)
ALTER PLUGGABLE DATABASE pdb_hr UNPLUG INTO '/u01/exports/pdb_hr.xml';

-- Drop the PDB from this CDB (keep datafiles!)
DROP PLUGGABLE DATABASE pdb_hr KEEP DATAFILES;

-- ═══════════════════════════════════════
-- PLUG a PDB into another CDB
-- ═══════════════════════════════════════

-- Check compatibility first
DECLARE
    l_result BOOLEAN;
BEGIN
    l_result := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(
        pdb_descr_file => '/u01/exports/pdb_hr.xml'
    );
    IF l_result THEN
        DBMS_OUTPUT.PUT_LINE('Compatible! ✅');
    ELSE
        DBMS_OUTPUT.PUT_LINE('NOT Compatible! ❌ Check PDB_PLUG_IN_VIOLATIONS');
    END IF;
END;
/

-- Check for violations
SELECT name, type, message, status
FROM pdb_plug_in_violations
WHERE status != 'RESOLVED';

-- Plug it in!
CREATE PLUGGABLE DATABASE pdb_hr USING '/u01/exports/pdb_hr.xml'
    FILE_NAME_CONVERT = ('/old_path/', '/new_path/')
    COPY;    -- COPY = copy datafiles | NOCOPY = use in place | MOVE = move files

-- Open and save state
ALTER PLUGGABLE DATABASE pdb_hr OPEN;
ALTER PLUGGABLE DATABASE pdb_hr SAVE STATE;
```

### PDB Management — Daily Operations

```sql
-- ═══════════════════════════════════════
-- SWITCHING BETWEEN CONTAINERS
-- ═══════════════════════════════════════

-- Show current container
SHOW CON_NAME;
-- or
SELECT SYS_CONTEXT('USERENV', 'CON_NAME') FROM dual;

-- Switch to CDB$ROOT
ALTER SESSION SET CONTAINER = CDB$ROOT;

-- Switch to a specific PDB
ALTER SESSION SET CONTAINER = PDB_HR;

-- ═══════════════════════════════════════
-- PDB STATUS MANAGEMENT
-- ═══════════════════════════════════════

-- View all PDBs and their status
SELECT con_id, name, open_mode, restricted, total_size / 1024 / 1024 AS size_mb
FROM v$pdbs
ORDER BY con_id;

-- Open all PDBs
ALTER PLUGGABLE DATABASE ALL OPEN;

-- Open specific PDB
ALTER PLUGGABLE DATABASE pdb_hr OPEN;

-- Close a PDB
ALTER PLUGGABLE DATABASE pdb_hr CLOSE IMMEDIATE;

-- Open PDB in READ ONLY mode
ALTER PLUGGABLE DATABASE pdb_hr OPEN READ ONLY;

-- Open PDB in RESTRICTED mode (only DBA can connect)
ALTER PLUGGABLE DATABASE pdb_hr OPEN RESTRICTED;

-- Save state (PDB auto-opens on CDB restart)
ALTER PLUGGABLE DATABASE pdb_hr SAVE STATE;

-- Discard saved state
ALTER PLUGGABLE DATABASE pdb_hr DISCARD STATE;

-- ═══════════════════════════════════════
-- PDB-LEVEL PARAMETERS
-- ═══════════════════════════════════════

-- Set parameter for specific PDB (while connected to that PDB)
ALTER SESSION SET CONTAINER = PDB_HR;
ALTER SYSTEM SET open_cursors = 500;
ALTER SYSTEM SET cursor_sharing = 'FORCE';

-- View PDB-specific parameters
SELECT name, value, ispdb_modifiable
FROM v$parameter
WHERE ispdb_modifiable = 'TRUE'
ORDER BY name;

-- ═══════════════════════════════════════
-- RENAME a PDB
-- ═══════════════════════════════════════

ALTER PLUGGABLE DATABASE pdb_hr CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE pdb_hr OPEN RESTRICTED;
ALTER PLUGGABLE DATABASE pdb_hr RENAME GLOBAL_NAME TO pdb_human_resources;
ALTER PLUGGABLE DATABASE pdb_human_resources CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE pdb_human_resources OPEN;

-- ═══════════════════════════════════════
-- DROP a PDB
-- ═══════════════════════════════════════

ALTER PLUGGABLE DATABASE pdb_hr CLOSE IMMEDIATE;
DROP PLUGGABLE DATABASE pdb_hr INCLUDING DATAFILES;
-- ⚠️ This permanently deletes the PDB and all its data!
```

---

## 🔐 Section 3: Users in Multitenant — Common vs Local

### The Two Types of Users

```
╔══════════════════════════════════════════════════════════════════════╗
║         COMMON USERS                    LOCAL USERS                 ║
║                                                                      ║
║   ┌─────────────────────┐          ┌─────────────────────┐         ║
║   │ C##DBA_ADMIN        │          │ APP_USER            │         ║
║   │                     │          │                     │         ║
║   │ ✅ Exists in ROOT   │          │ ❌ NOT in ROOT       │         ║
║   │ ✅ Exists in ALL    │          │ ✅ Exists in ONE     │         ║
║   │    PDBs             │          │    PDB only          │         ║
║   │                     │          │                     │         ║
║   │ Created in ROOT     │          │ Created inside PDB  │         ║
║   │ Name starts with    │          │ Normal name          │         ║
║   │ C## (mandatory)     │          │ (no C## prefix)     │         ║
║   │                     │          │                     │         ║
║   │ Use for: DBAs,      │          │ Use for: App users, │         ║
║   │ cross-PDB admin     │          │ developers,         │         ║
║   │                     │          │ app schemas          │         ║
║   └─────────────────────┘          └─────────────────────┘         ║
╚══════════════════════════════════════════════════════════════════════╝
```

```sql
-- ═══════════════════════════════════════
-- COMMON USER (created in ROOT, visible everywhere)
-- ═══════════════════════════════════════

-- Connect to ROOT
ALTER SESSION SET CONTAINER = CDB$ROOT;

CREATE USER c##global_dba IDENTIFIED BY "SecureP@ss!"
    CONTAINER = ALL;    -- Exists in root + all PDBs

GRANT DBA TO c##global_dba CONTAINER = ALL;
-- This user can manage ALL PDBs from a single login

-- ═══════════════════════════════════════
-- LOCAL USER (created in PDB, visible only there)
-- ═══════════════════════════════════════

-- Connect to specific PDB
ALTER SESSION SET CONTAINER = PDB_HR;

CREATE USER hr_app_user IDENTIFIED BY "AppP@ss123!"
    DEFAULT TABLESPACE users
    QUOTA 1G ON users;

GRANT CREATE SESSION, CREATE TABLE TO hr_app_user;
-- This user exists ONLY in PDB_HR

-- ═══════════════════════════════════════
-- CONNECTING TO A SPECIFIC PDB (from client)
-- ═══════════════════════════════════════

-- TNS entry (tnsnames.ora)
-- PDB_HR =
--   (DESCRIPTION =
--     (ADDRESS = (PROTOCOL = TCP)(HOST = dbserver)(PORT = 1521))
--     (CONNECT_DATA =
--       (SERVER = DEDICATED)
--       (SERVICE_NAME = pdb_hr)     ← PDB service name
--     )
--   )

-- Easy Connect syntax
-- sqlplus hr_app_user/password@dbserver:1521/pdb_hr
```

---

## ⚖️ Section 4: Resource Management — PDB Isolation

### The Problem

Without resource management, one PDB can hog all the CPU and memory, starving the others:

```
╔══════════════════════════════════════════════════════════════════╗
║         WITHOUT RESOURCE MANAGEMENT                              ║
║                                                                  ║
║   PDB_HR:  "I need 95% of CPU for my batch job!" 🐷            ║
║   PDB_FIN: "I can't even run a SELECT..." 😢                   ║
║   PDB_CRM: "Users are complaining about timeouts!" 😤          ║
║                                                                  ║
║         WITH RESOURCE MANAGEMENT                                 ║
║                                                                  ║
║   PDB_HR:  "I get 40% CPU max" → Batch runs in its lane        ║
║   PDB_FIN: "I get 35% CPU guaranteed" → Always responsive      ║
║   PDB_CRM: "I get 25% CPU guaranteed" → Users are happy        ║
╚══════════════════════════════════════════════════════════════════╝
```

### CDB Resource Plan

```sql
-- Create a CDB resource plan to limit PDB resource usage

-- Step 1: Create the plan
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN(
        plan    => 'CDB_RESOURCE_PLAN',
        comment => 'Control resources across PDBs'
    );
END;
/

-- Step 2: Create plan directives for each PDB
BEGIN
    -- HR gets up to 40% CPU, guaranteed 20%
    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN_DIRECTIVE(
        plan                  => 'CDB_RESOURCE_PLAN',
        pluggable_database    => 'PDB_HR',
        shares                => 4,         -- Relative weight (40%)
        utilization_limit     => 60,        -- Max 60% of total CPU
        parallel_server_limit => 50         -- Max 50% of PQ slaves
    );

    -- Finance gets 35%, guaranteed
    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN_DIRECTIVE(
        plan                  => 'CDB_RESOURCE_PLAN',
        pluggable_database    => 'PDB_FIN',
        shares                => 3,
        utilization_limit     => 50,
        parallel_server_limit => 30
    );

    -- Default directive for other PDBs
    DBMS_RESOURCE_MANAGER.UPDATE_CDB_DEFAULT_DIRECTIVE(
        plan              => 'CDB_RESOURCE_PLAN',
        new_shares        => 1,
        new_utilization_limit => 30,
        new_parallel_server_limit => 20
    );
END;
/

-- Step 3: Activate the plan
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'CDB_RESOURCE_PLAN';
```

### PDB-Level Memory Limits

```sql
-- Set memory limits for a specific PDB
ALTER SESSION SET CONTAINER = PDB_HR;

-- Limit SGA usage (19c+)
ALTER SYSTEM SET SGA_TARGET = 4G;           -- PDB can use up to 4GB SGA
ALTER SYSTEM SET SGA_MIN_SIZE = 2G;         -- Guaranteed 2GB SGA

-- Limit PGA usage
ALTER SYSTEM SET PGA_AGGREGATE_LIMIT = 2G;
ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 1G;

-- These are SOFT limits — Oracle Resource Manager enforces them
```

### PDB-Level I/O Limits

```sql
-- Limit I/O per PDB (Oracle 12.2+)
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN_DIRECTIVE(
        plan               => 'IO_PLAN',
        pluggable_database => 'PDB_DEV',
        shares             => 1,
        utilization_limit  => 20,     -- Max 20% CPU
        -- I/O limits are controlled via shares (proportional)
        parallel_server_limit => 10
    );
END;
/

-- View current resource allocation
SELECT
    r.con_id,
    p.name AS pdb_name,
    r.shares,
    r.utilization_limit,
    r.parallel_server_limit
FROM v$rsrc_plan_directive r
JOIN v$pdbs p ON r.con_id = p.con_id
ORDER BY r.con_id;
```

---

## 💾 Section 5: PDB Backup, Recovery & Flashback

### PDB-Level RMAN Backup

```sql
-- ═══════════════════════════════════════
-- BACKUP A SPECIFIC PDB
-- ═══════════════════════════════════════

RMAN> BACKUP PLUGGABLE DATABASE pdb_hr
        TAG 'PDB_HR_FULL';

RMAN> BACKUP INCREMENTAL LEVEL 1 PLUGGABLE DATABASE pdb_hr
        TAG 'PDB_HR_INCR';

-- ═══════════════════════════════════════
-- BACKUP ALL PDBs (whole CDB)
-- ═══════════════════════════════════════

RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
-- This backs up CDB$ROOT + PDB$SEED + ALL PDBs

-- ═══════════════════════════════════════
-- RECOVER A SPECIFIC PDB (without affecting others!)
-- ═══════════════════════════════════════

-- Close the PDB
ALTER PLUGGABLE DATABASE pdb_hr CLOSE IMMEDIATE;

RMAN> RESTORE PLUGGABLE DATABASE pdb_hr;
RMAN> RECOVER PLUGGABLE DATABASE pdb_hr;

ALTER PLUGGABLE DATABASE pdb_hr OPEN RESETLOGS;

-- ═══════════════════════════════════════
-- PDB POINT-IN-TIME RECOVERY (PITR)
-- ═══════════════════════════════════════

-- ⚠️ Requires LOCAL UNDO MODE (per-PDB undo tablespace)

-- Close the PDB
ALTER PLUGGABLE DATABASE pdb_hr CLOSE IMMEDIATE;

RMAN> -- Recover PDB_HR to 2 hours ago
RMAN> RUN {
        SET UNTIL TIME "TO_DATE('2026-06-02 14:00:00', 'YYYY-MM-DD HH24:MI:SS')";
        RESTORE PLUGGABLE DATABASE pdb_hr;
        RECOVER PLUGGABLE DATABASE pdb_hr;
      }

ALTER PLUGGABLE DATABASE pdb_hr OPEN RESETLOGS;
-- Only PDB_HR is affected — all other PDBs stay online! 🎯
```

### PDB Flashback

```sql
-- Flash back JUST one PDB to a previous point in time
-- (Requires Flashback Database enabled + Local Undo Mode)

ALTER PLUGGABLE DATABASE pdb_hr CLOSE IMMEDIATE;

FLASHBACK PLUGGABLE DATABASE pdb_hr TO TIMESTAMP
    (SYSTIMESTAMP - INTERVAL '2' HOUR);

ALTER PLUGGABLE DATABASE pdb_hr OPEN RESETLOGS;

-- Other PDBs are COMPLETELY UNAFFECTED!

-- Create a PDB restore point (like a snapshot)
CREATE RESTORE POINT pdb_hr_before_upgrade
    FOR PLUGGABLE DATABASE pdb_hr
    GUARANTEE FLASHBACK DATABASE;

-- Flash back to the restore point
FLASHBACK PLUGGABLE DATABASE pdb_hr TO RESTORE POINT pdb_hr_before_upgrade;
```

---

## 📊 Section 6: PDB Snapshots & Cloning (19c+)

### PDB Snapshot Carousel — Automatic Snapshots

```sql
-- Create automatic snapshots every 30 minutes (up to 8 snapshots)
ALTER PLUGGABLE DATABASE pdb_hr SET
    MAX_PDB_SNAPSHOTS = 8;

ALTER PLUGGABLE DATABASE pdb_hr SNAPSHOT MODE EVERY 30 MINUTES;

-- View available snapshots
SELECT con_id, snapshot_name, snapshot_scn, snapshot_time
FROM dba_pdb_snapshots
WHERE con_id = (SELECT con_id FROM v$pdbs WHERE name = 'PDB_HR');

-- Create a PDB from a snapshot (instant thin clone!)
CREATE PLUGGABLE DATABASE pdb_hr_debug FROM pdb_hr
    USING SNAPSHOT AT SCN 123456789;

-- Refresh a cloned PDB (pull latest changes from source)
ALTER PLUGGABLE DATABASE pdb_hr_dev CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE pdb_hr_dev REFRESH;
ALTER PLUGGABLE DATABASE pdb_hr_dev OPEN;
```

### Refreshable Clone PDBs

```sql
-- Create a PDB that auto-refreshes from the source (for dev/test)
CREATE PLUGGABLE DATABASE pdb_hr_test FROM pdb_hr
    REFRESH MODE EVERY 60 MINUTES;    -- Auto-refresh hourly

-- The test PDB always has near-real-time data from production!
-- Great for QA testing against real data.

-- Convert a refreshable clone to a standalone PDB
ALTER PLUGGABLE DATABASE pdb_hr_test REFRESH MODE NONE;
ALTER PLUGGABLE DATABASE pdb_hr_test OPEN;
```

---

## 🔍 Section 7: Monitoring the Multitenant Environment

### Essential Multitenant Views

```sql
-- ═══════════════════════════════════════
-- CDB-WIDE MONITORING (from ROOT)
-- ═══════════════════════════════════════

-- All PDBs and their status
SELECT
    con_id, name, open_mode, restricted,
    ROUND(total_size / 1024 / 1024 / 1024, 2) AS size_gb
FROM v$pdbs
ORDER BY con_id;

-- Data dictionary across all PDBs (CDB_ views)
-- CDB_ prefix = same as DBA_ but includes CON_ID column
SELECT con_id, tablespace_name, 
       ROUND(SUM(bytes) / 1024 / 1024 / 1024, 2) AS size_gb
FROM cdb_data_files
GROUP BY con_id, tablespace_name
ORDER BY con_id, tablespace_name;

-- Users across all PDBs
SELECT con_id, username, account_status, common
FROM cdb_users
WHERE oracle_maintained = 'N'
ORDER BY con_id, username;

-- Active sessions across all PDBs
SELECT
    s.con_id,
    p.name AS pdb_name,
    s.sid, s.serial#, s.username,
    s.sql_id, s.event, s.status
FROM v$session s
JOIN v$pdbs p ON s.con_id = p.con_id
WHERE s.type = 'USER' AND s.status = 'ACTIVE';

-- PDB resource usage
SELECT
    r.con_id,
    p.name AS pdb_name,
    r.cpu_consumed_time / 1000 AS cpu_seconds,
    r.sga_bytes / 1024 / 1024 AS sga_mb,
    r.pga_bytes / 1024 / 1024 AS pga_mb,
    r.io_service_time / 1000 AS io_seconds
FROM v$rsrc_consumer_group r
JOIN v$pdbs p ON r.con_id = p.con_id
WHERE r.con_id > 2;  -- Exclude ROOT and SEED

-- PDB history (startup/shutdown/open/close events)
SELECT name, cause, timestamp, pdb_incarnation#
FROM cdb_pdb_history
ORDER BY timestamp DESC;
```

---

## 🧪 Interview-Ready Explanations

### "Explain Oracle Multitenant in 30 seconds"

> *"Oracle Multitenant lets you run multiple pluggable databases (PDBs) inside one container database (CDB). Each PDB looks like a standalone database to the application, but they all share a single set of background processes, memory, and redo logs. This means one patch cycle, one RMAN config, and one set of background processes for dozens of databases. PDBs can be cloned, unplugged, and plugged into different CDBs like USB drives."*

### Quick-Fire Interview Questions

| Question | Answer |
|----------|--------|
| "What's a CDB?" | Container Database — the master container that holds the root, seed, and all pluggable databases. |
| "CDB$ROOT vs PDB$SEED?" | ROOT = data dictionary master + common users. SEED = read-only template used to create new PDBs. |
| "Common vs Local user?" | Common user (C## prefix) exists in ROOT + all PDBs. Local user exists in one PDB only. App users should be local. |
| "Can you patch one PDB?" | No. Patching is at the CDB level — all PDBs get patched together. This is actually an advantage (one patch cycle). |
| "PDB PITR?" | Yes! With Local Undo Mode, you can point-in-time recover a single PDB without affecting others. |
| "Max PDBs per CDB?" | Oracle supports up to 4,096 PDBs per CDB (practical limit depends on hardware). |
| "Unplug/Plug?" | Unplug exports PDB metadata to XML. Plug imports it into another CDB. Datafiles are copied or moved. Like a USB drive for databases. |
| "What is Local Undo?" | Each PDB has its own undo tablespace (vs shared undo). Required for PDB-level flashback and PITR. |

---

## ⚠️ Common Misconceptions

```
╔════════════════════════════════════════════════════════════════════════╗
║ MYTH                             │ REALITY                           ║
╠══════════════════════════════════╪═══════════════════════════════════╣
║ "Multitenant is just             │ It's a fundamental architecture   ║
║  database consolidation"         │ change. PDBs are portable,        ║
║                                  │ clonable, and independently       ║
║                                  │ recoverable.                      ║
╠══════════════════════════════════╪═══════════════════════════════════╣
║ "PDBs share data with           │ COMPLETE data isolation. PDB_HR   ║
║  each other"                     │ cannot see PDB_FIN's tables.      ║
║                                  │ They're logically separate DBs.   ║
╠══════════════════════════════════╪═══════════════════════════════════╣
║ "One PDB crash brings           │ A PDB can be closed/recovered     ║
║  down all PDBs"                  │ independently. Other PDBs stay up.║
╠══════════════════════════════════╪═══════════════════════════════════╣
║ "Multitenant is only for        │ As of 21c, non-CDB architecture   ║
║  large enterprises"              │ is deprecated. Even single-DB     ║
║                                  │ setups use CDB with 1 PDB.        ║
╠══════════════════════════════════╪═══════════════════════════════════╣
║ "Performance is worse            │ Overhead is negligible (<2%).     ║
║  with Multitenant"               │ Shared memory can actually        ║
║                                  │ IMPROVE efficiency.               ║
╚══════════════════════════════════╧═══════════════════════════════════╝
```

---

## 🔑 Key Takeaways

```
✅ CDB = Container (one instance, shared resources)
✅ PDB = Pluggable Database (looks standalone to the app)
✅ CDB$ROOT = metadata master | PDB$SEED = template for new PDBs
✅ Common users (C##) span all PDBs | Local users live in one PDB
✅ Unplug/Plug = portability — move databases between CDBs like USB drives
✅ Resource plans control CPU, memory, I/O per PDB — no noisy neighbors
✅ PDB-level backup, recovery, and flashback (with Local Undo Mode)
✅ Refreshable clone PDBs = always-fresh dev/test environments
✅ PDB Snapshot Carousel = automatic thin clones on schedule
✅ Non-CDB architecture is DEPRECATED from 21c — Multitenant is the future
✅ Max 4,096 PDBs per CDB — consolidate everything
✅ One patch cycle, one RMAN config, one set of processes = massive savings
```

---

## 🔗 What's Next?

You've completed the **Oracle Database** track! 🎉

**Next up → Chapter 2C: [Microsoft SQL Server](../04-SQL-Server/01-SQLServer-Architecture.md)**
Where we explore the **powerhouse of the Microsoft ecosystem** — T-SQL, Always On, SSMS, and Azure SQL.

---

> *"Multitenant is not just a feature — it's the future of Oracle. Every Oracle database will be a pluggable database."* — Oracle Corporation
