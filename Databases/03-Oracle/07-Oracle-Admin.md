# 🛠️ Chapter 2B.7 — Oracle Administration & Maintenance

> **Level:** 🔴 Advanced
> **Time to Master:** ~6-8 hours
> **Prerequisites:** Chapter 2B.1 (Oracle Architecture), Chapter 2B.2 (Installation)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Manage **Tablespaces** — the storage containers of Oracle
- Create and administer **Users, Roles, and Privileges** like a security pro
- Perform **RMAN Backups** — the gold standard for Oracle recovery
- Use **Data Pump** (expdp/impdp) for blazing-fast exports and imports
- Plan and execute **Oracle patching** without sweating

---

## 🧠 The Big Question

> *"So we've built the database, tuned it, and set up HA. Now... who keeps it alive day-to-day?"*

That's the **DBA** — the Database Administrator. And this chapter is your DBA survival manual. These are the tasks that happen **every single day** in production:

```
╔══════════════════════════════════════════════════════════════════════╗
║                   A DBA'S TYPICAL WEEK                               ║
║                                                                      ║
║   Monday    → Check backup status, review alert logs                ║
║   Tuesday   → Add tablespace, create new schema user               ║
║   Wednesday → Investigate space issues, gather stats                ║
║   Thursday  → Plan quarterly patching window                        ║
║   Friday    → Export/import for dev refresh, audit reviews          ║
║   Saturday  → RMAN backup verification, test restore               ║
║   Sunday    → 😴 (unless pager goes off at 3 AM)                   ║
║                                                                      ║
║   Every day → Monitor, monitor, monitor                             ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🗄️ Section 1: Tablespaces — Where Data Lives

### What is a Tablespace?

A **tablespace** is a logical storage container in Oracle. Every table, index, and object lives inside a tablespace. A tablespace consists of one or more **datafiles** on disk.

```
╔══════════════════════════════════════════════════════════════════╗
║              ORACLE STORAGE HIERARCHY                            ║
║                                                                  ║
║   DATABASE                                                       ║
║   └── TABLESPACE (logical container)                             ║
║       └── SEGMENT (table, index, etc.)                           ║
║           └── EXTENT (contiguous blocks)                         ║
║               └── DATA BLOCK (smallest unit, default 8KB)        ║
║                                                                  ║
║   On Disk:                                                       ║
║   TABLESPACE → DATAFILE(s) → OS file on filesystem/ASM          ║
║                                                                  ║
║   Example:                                                       ║
║   ┌─────────────────────────────────────────┐                   ║
║   │  TABLESPACE: USERS                       │                   ║
║   │  ┌──────────────┐  ┌──────────────┐     │                   ║
║   │  │ users01.dbf  │  │ users02.dbf  │     │                   ║
║   │  │   (10 GB)    │  │   (10 GB)    │     │                   ║
║   │  └──────────────┘  └──────────────┘     │                   ║
║   │                                          │                   ║
║   │  Contains: EMPLOYEES table, EMP_IDX      │                   ║
║   │            DEPARTMENTS table, etc.       │                   ║
║   └─────────────────────────────────────────┘                   ║
╚══════════════════════════════════════════════════════════════════╝
```

### Default Tablespaces Every Oracle DB Has

```
╔═════════════════════════════════════════════════════════════════╗
║  Tablespace    │ Purpose                                       ║
╠════════════════╪═══════════════════════════════════════════════╣
║  SYSTEM        │ Data dictionary (Oracle's internal metadata)  ║
║                │ ⚠️ NEVER put user data here!                  ║
╠════════════════╪═══════════════════════════════════════════════╣
║  SYSAUX        │ Auxiliary system data (AWR, ADDM, etc.)       ║
║                │ ⚠️ NEVER put user data here!                  ║
╠════════════════╪═══════════════════════════════════════════════╣
║  UNDOTBS1      │ Undo data (for rollback and read consistency) ║
╠════════════════╪═══════════════════════════════════════════════╣
║  TEMP          │ Temporary data (sorts, hash joins, etc.)      ║
╠════════════════╪═══════════════════════════════════════════════╣
║  USERS         │ Default tablespace for user data              ║
╚════════════════╧═══════════════════════════════════════════════╝
```

### Tablespace Management — Full Command Reference

```sql
-- ═══════════════════════════════════════
-- CREATING TABLESPACES
-- ═══════════════════════════════════════

-- Create a basic tablespace
CREATE TABLESPACE app_data
    DATAFILE '/u01/oradata/PROD/app_data01.dbf' SIZE 10G
    AUTOEXTEND ON NEXT 1G MAXSIZE 50G
    EXTENT MANAGEMENT LOCAL             -- Always use local (modern Oracle)
    SEGMENT SPACE MANAGEMENT AUTO;      -- Let Oracle manage free space

-- Create a BIGFILE tablespace (single large file — simpler management)
CREATE BIGFILE TABLESPACE big_data
    DATAFILE '/u01/oradata/PROD/big_data01.dbf' SIZE 100G
    AUTOEXTEND ON NEXT 10G MAXSIZE UNLIMITED;

-- Create a temporary tablespace
CREATE TEMPORARY TABLESPACE temp_large
    TEMPFILE '/u01/oradata/PROD/temp_large01.dbf' SIZE 20G
    AUTOEXTEND ON NEXT 5G MAXSIZE 100G;

-- Create an UNDO tablespace
CREATE UNDO TABLESPACE undotbs2
    DATAFILE '/u01/oradata/PROD/undotbs2_01.dbf' SIZE 10G
    AUTOEXTEND ON NEXT 1G MAXSIZE 50G;

-- ═══════════════════════════════════════
-- MODIFYING TABLESPACES
-- ═══════════════════════════════════════

-- Add a datafile (when tablespace is running out of space)
ALTER TABLESPACE app_data
    ADD DATAFILE '/u01/oradata/PROD/app_data02.dbf' SIZE 10G
    AUTOEXTEND ON NEXT 1G MAXSIZE 50G;

-- Resize an existing datafile
ALTER DATABASE DATAFILE '/u01/oradata/PROD/app_data01.dbf'
    RESIZE 20G;

-- Enable autoextend on an existing datafile
ALTER DATABASE DATAFILE '/u01/oradata/PROD/app_data01.dbf'
    AUTOEXTEND ON NEXT 1G MAXSIZE 50G;

-- Take tablespace OFFLINE (for maintenance)
ALTER TABLESPACE app_data OFFLINE;
ALTER TABLESPACE app_data ONLINE;

-- Make tablespace READ-ONLY (for archival)
ALTER TABLESPACE app_data READ ONLY;
ALTER TABLESPACE app_data READ WRITE;

-- ═══════════════════════════════════════
-- MONITORING TABLESPACES
-- ═══════════════════════════════════════

-- Check tablespace usage (THE query every DBA runs daily ⭐)
SELECT
    df.tablespace_name,
    ROUND(df.total_mb, 2)                     AS total_mb,
    ROUND(df.total_mb - NVL(fs.free_mb, 0), 2) AS used_mb,
    ROUND(NVL(fs.free_mb, 0), 2)               AS free_mb,
    ROUND((df.total_mb - NVL(fs.free_mb, 0)) / df.total_mb * 100, 1) AS pct_used
FROM (
    SELECT tablespace_name, SUM(bytes) / 1024 / 1024 AS total_mb
    FROM dba_data_files
    GROUP BY tablespace_name
) df
LEFT JOIN (
    SELECT tablespace_name, SUM(bytes) / 1024 / 1024 AS free_mb
    FROM dba_free_space
    GROUP BY tablespace_name
) fs ON df.tablespace_name = fs.tablespace_name
ORDER BY pct_used DESC;

-- Check datafile sizes and autoextend
SELECT
    file_name,
    tablespace_name,
    ROUND(bytes / 1024 / 1024) AS size_mb,
    ROUND(maxbytes / 1024 / 1024) AS max_mb,
    autoextensible
FROM dba_data_files
ORDER BY tablespace_name, file_name;

-- ═══════════════════════════════════════
-- DROPPING TABLESPACES
-- ═══════════════════════════════════════

-- Drop tablespace (data must be empty or use CASCADE)
DROP TABLESPACE app_data INCLUDING CONTENTS AND DATAFILES;
-- ⚠️ DANGEROUS: Permanently deletes ALL data and files!
```

### ASM — Automatic Storage Management

```
╔══════════════════════════════════════════════════════════════════╗
║                  ASM (Automatic Storage Management)              ║
║                                                                  ║
║   Instead of manually managing OS files, ASM manages disks      ║
║   directly — like a volume manager + filesystem + RAID in one.  ║
║                                                                  ║
║   ┌────────────────────────────────────────┐                    ║
║   │         ASM DISK GROUP: +DATA         │                    ║
║   │                                        │                    ║
║   │  ┌────────┐ ┌────────┐ ┌────────┐    │                    ║
║   │  │ Disk 1 │ │ Disk 2 │ │ Disk 3 │    │                    ║
║   │  │ 500GB  │ │ 500GB  │ │ 500GB  │    │                    ║
║   │  └────────┘ └────────┘ └────────┘    │                    ║
║   │                                        │                    ║
║   │  Redundancy: NORMAL (2-way mirror)     │                    ║
║   │  Oracle distributes data & stripes     │                    ║
║   │  automatically across all disks        │                    ║
║   └────────────────────────────────────────┘                    ║
║                                                                  ║
║   Benefits:                                                      ║
║   ✅ No OS filesystem needed                                    ║
║   ✅ Automatic rebalancing when disks added/removed             ║
║   ✅ Built-in mirroring (replaces RAID)                         ║
║   ✅ Optimized I/O for Oracle                                   ║
╚══════════════════════════════════════════════════════════════════╝
```

```sql
-- ASM disk group commands (run in ASM instance)
-- Create disk group
CREATE DISKGROUP data_dg
    NORMAL REDUNDANCY
    DISK '/dev/sdb1', '/dev/sdc1', '/dev/sdd1', '/dev/sde1';

-- Add disks (ASM auto-rebalances)
ALTER DISKGROUP data_dg ADD DISK '/dev/sdf1';

-- Check ASM disk group space
SELECT
    name,
    total_mb,
    free_mb,
    ROUND((total_mb - free_mb) / total_mb * 100, 1) AS pct_used,
    type AS redundancy
FROM v$asm_diskgroup;

-- Create tablespace on ASM (use +DISKGROUP_NAME)
CREATE TABLESPACE app_data
    DATAFILE '+DATA' SIZE 10G
    AUTOEXTEND ON NEXT 1G MAXSIZE 50G;
```

---

## 👥 Section 2: Users, Roles & Privileges

### Oracle Security Model

```
╔══════════════════════════════════════════════════════════════════╗
║              ORACLE SECURITY HIERARCHY                           ║
║                                                                  ║
║   USER (schema owner)                                            ║
║   ├── Has a DEFAULT TABLESPACE                                   ║
║   ├── Has a TEMPORARY TABLESPACE                                 ║
║   ├── Has a PROFILE (password policies, resource limits)         ║
║   │                                                              ║
║   ├── SYSTEM PRIVILEGES (granted directly)                       ║
║   │   └── CREATE SESSION, CREATE TABLE, etc.                     ║
║   │                                                              ║
║   ├── OBJECT PRIVILEGES (granted on specific objects)            ║
║   │   └── SELECT ON hr.employees, INSERT ON hr.orders            ║
║   │                                                              ║
║   └── ROLES (bundles of privileges)                              ║
║       ├── CONNECT (basic login)                                  ║
║       ├── RESOURCE (create objects)                               ║
║       ├── DBA (everything)                                       ║
║       └── Custom roles (APP_READONLY, APP_ADMIN, etc.)           ║
╚══════════════════════════════════════════════════════════════════╝
```

### User Management

```sql
-- ═══════════════════════════════════════
-- CREATING USERS
-- ═══════════════════════════════════════

-- Create a basic user
CREATE USER app_user IDENTIFIED BY "SecureP@ss123!"
    DEFAULT TABLESPACE app_data
    TEMPORARY TABLESPACE temp
    QUOTA 5G ON app_data             -- Space limit on tablespace
    QUOTA 0 ON system                -- NO space on SYSTEM (important!)
    PROFILE default;

-- Create a user with unlimited quota
CREATE USER admin_user IDENTIFIED BY "Adm1nP@ss!"
    DEFAULT TABLESPACE users
    TEMPORARY TABLESPACE temp
    QUOTA UNLIMITED ON users;

-- ═══════════════════════════════════════
-- GRANTING PRIVILEGES
-- ═══════════════════════════════════════

-- System privileges (what can they DO in the database?)
GRANT CREATE SESSION TO app_user;          -- Login
GRANT CREATE TABLE TO app_user;            -- Create tables
GRANT CREATE VIEW TO app_user;             -- Create views
GRANT CREATE SEQUENCE TO app_user;         -- Create sequences
GRANT CREATE PROCEDURE TO app_user;        -- Create PL/SQL

-- Object privileges (what can they access?)
GRANT SELECT ON hr.employees TO app_user;
GRANT INSERT, UPDATE ON hr.orders TO app_user;
GRANT EXECUTE ON hr.calculate_bonus TO app_user;
GRANT SELECT ON hr.employees TO app_user WITH GRANT OPTION;
-- WITH GRANT OPTION = app_user can grant this to others

-- ═══════════════════════════════════════
-- ROLES — Bundle privileges for easy management
-- ═══════════════════════════════════════

-- Create custom roles (BEST PRACTICE ⭐)
CREATE ROLE app_readonly;
GRANT CREATE SESSION TO app_readonly;
GRANT SELECT ON hr.employees TO app_readonly;
GRANT SELECT ON hr.departments TO app_readonly;
GRANT SELECT ON hr.orders TO app_readonly;

CREATE ROLE app_readwrite;
GRANT app_readonly TO app_readwrite;   -- Inherit read permissions
GRANT INSERT, UPDATE, DELETE ON hr.orders TO app_readwrite;

CREATE ROLE app_admin;
GRANT app_readwrite TO app_admin;      -- Inherit read + write
GRANT CREATE TABLE, CREATE VIEW, CREATE SEQUENCE TO app_admin;
GRANT ALTER ANY TABLE TO app_admin;

-- Assign roles to users
GRANT app_readonly TO report_user;
GRANT app_readwrite TO app_user;
GRANT app_admin TO admin_user;

-- ═══════════════════════════════════════
-- MODIFYING USERS
-- ═══════════════════════════════════════

-- Change password
ALTER USER app_user IDENTIFIED BY "NewSecureP@ss456!";

-- Lock/unlock account
ALTER USER app_user ACCOUNT LOCK;
ALTER USER app_user ACCOUNT UNLOCK;

-- Change quota
ALTER USER app_user QUOTA 10G ON app_data;

-- Change default tablespace
ALTER USER app_user DEFAULT TABLESPACE new_tablespace;

-- ═══════════════════════════════════════
-- REVOKING PRIVILEGES
-- ═══════════════════════════════════════

REVOKE INSERT ON hr.orders FROM app_user;
REVOKE app_readwrite FROM app_user;

-- ═══════════════════════════════════════
-- VIEWING USER INFORMATION
-- ═══════════════════════════════════════

-- All users
SELECT username, account_status, default_tablespace, 
       created, profile, authentication_type
FROM dba_users
WHERE oracle_maintained = 'N'    -- Exclude Oracle internal users
ORDER BY created DESC;

-- User's privileges
SELECT * FROM dba_sys_privs WHERE grantee = 'APP_USER';
SELECT * FROM dba_tab_privs WHERE grantee = 'APP_USER';
SELECT * FROM dba_role_privs WHERE grantee = 'APP_USER';

-- What's in a role?
SELECT * FROM dba_sys_privs WHERE grantee = 'APP_READONLY';
SELECT * FROM dba_tab_privs WHERE grantee = 'APP_READONLY';

-- Drop user (and everything they own)
DROP USER app_user CASCADE;
-- ⚠️ CASCADE drops ALL objects owned by the user!
```

### Profiles — Password Policies & Resource Limits

```sql
-- Create a profile with strong password policies
CREATE PROFILE secure_profile LIMIT
    FAILED_LOGIN_ATTEMPTS 5              -- Lock after 5 bad passwords
    PASSWORD_LOCK_TIME 1/24              -- Lock for 1 hour (1/24 of a day)
    PASSWORD_LIFE_TIME 90                -- Password expires in 90 days
    PASSWORD_GRACE_TIME 7                -- 7-day grace period
    PASSWORD_REUSE_TIME 365              -- Can't reuse password for 1 year
    PASSWORD_REUSE_MAX 12                -- Must use 12 different passwords
    PASSWORD_VERIFY_FUNCTION ora12c_verify_function  -- Complexity check
    SESSIONS_PER_USER 5                  -- Max 5 concurrent sessions
    IDLE_TIME 30                         -- Disconnect after 30 min idle
    CONNECT_TIME 480;                    -- Max 8 hours per session

-- Assign profile to user
ALTER USER app_user PROFILE secure_profile;

-- View profiles
SELECT * FROM dba_profiles WHERE profile = 'SECURE_PROFILE';
```

---

## 💾 Section 3: RMAN Backup & Recovery — Your Safety Net

### What is RMAN?

**Recovery Manager (RMAN)** is Oracle's dedicated backup and recovery tool. It's infinitely better than manual OS-level copies because it:
- Knows which blocks are **actually used** (skips empty blocks)
- Can do **incremental backups** (only changed blocks)
- Handles **redo log archiving** automatically
- Can do **block-level recovery** (fix one corrupted block without restoring the whole file)

```
╔══════════════════════════════════════════════════════════════════╗
║                 RMAN BACKUP TYPES                                ║
║                                                                  ║
║   ┌─────────────┐   ┌─────────────┐   ┌──────────────────┐    ║
║   │   FULL      │   │ INCREMENTAL │   │  INCREMENTAL     │    ║
║   │   BACKUP    │   │  LEVEL 0    │   │   LEVEL 1        │    ║
║   │             │   │  (= Full)   │   │   (changes only) │    ║
║   │ All blocks  │   │  Base for   │   │                  │    ║
║   │ 100 GB      │   │  incrementals│   │  Only changed   │    ║
║   │ Time: 2 hrs │   │  100 GB     │   │  blocks: 5 GB   │    ║
║   │             │   │  Time: 2 hrs│   │  Time: 15 min   │    ║
║   └─────────────┘   └─────────────┘   └──────────────────┘    ║
║                                                                  ║
║   BEST PRACTICE ⭐:                                             ║
║   Sunday: Level 0 (full base)                                    ║
║   Mon-Sat: Level 1 incremental (fast, small)                    ║
║   Every hour: Archive log backup                                 ║
║                                                                  ║
║   Recovery = Level 0 + Level 1(s) + Archive logs                ║
║            = Point-in-time recovery possible! ⏰                ║
╚══════════════════════════════════════════════════════════════════╝
```

### RMAN Commands — Essential Toolkit

```sql
-- ═══════════════════════════════════════
-- CONNECTING TO RMAN
-- ═══════════════════════════════════════

-- Connect to target database
rman target /

-- Connect to target with catalog (recommended for production)
rman target / catalog rman_user/password@rman_catalog

-- ═══════════════════════════════════════
-- CONFIGURING RMAN
-- ═══════════════════════════════════════

RMAN> -- Set backup retention policy (keep 7 days)
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;

RMAN> -- Set default backup location (Fast Recovery Area)
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK
        FORMAT '/u01/backups/PROD/%U';

RMAN> -- Enable backup compression (saves 60-70% space)
RMAN> CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO COMPRESSED BACKUPSET;

RMAN> -- Enable block change tracking (speeds up incrementals 10x)
RMAN> ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
        USING FILE '/u01/oradata/PROD/block_change_tracking.dbf';

RMAN> -- Auto-backup controlfile and SPFILE
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;

RMAN> -- Set parallelism (use multiple channels)
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 4;

RMAN> -- Show all settings
RMAN> SHOW ALL;

-- ═══════════════════════════════════════
-- BACKUP COMMANDS
-- ═══════════════════════════════════════

RMAN> -- Full database backup
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

RMAN> -- Incremental Level 0 (base)
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;

RMAN> -- Incremental Level 1 (changes since Level 0)
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;

RMAN> -- Backup specific tablespace
RMAN> BACKUP TABLESPACE users, app_data;

RMAN> -- Backup specific datafile
RMAN> BACKUP DATAFILE 4;

RMAN> -- Backup archive logs only
RMAN> BACKUP ARCHIVELOG ALL;

RMAN> -- Backup and delete old archive logs
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;

RMAN> -- Compressed backup with tag
RMAN> BACKUP AS COMPRESSED BACKUPSET
        DATABASE
        TAG 'WEEKLY_FULL_2026_06_01'
        PLUS ARCHIVELOG;

-- ═══════════════════════════════════════
-- PRODUCTION BACKUP SCRIPT (run via cron/scheduler)
-- ═══════════════════════════════════════

-- Daily incremental script
RUN {
    ALLOCATE CHANNEL ch1 DEVICE TYPE DISK;
    ALLOCATE CHANNEL ch2 DEVICE TYPE DISK;
    ALLOCATE CHANNEL ch3 DEVICE TYPE DISK;
    ALLOCATE CHANNEL ch4 DEVICE TYPE DISK;

    BACKUP INCREMENTAL LEVEL 1
        CUMULATIVE                      -- Include ALL changes since Level 0
        DATABASE
        TAG 'DAILY_INCR'
        FORMAT '/u01/backups/PROD/%d_%T_%U';

    BACKUP ARCHIVELOG ALL
        TAG 'DAILY_ARCHLOG'
        DELETE INPUT;                   -- Remove archived logs after backup

    -- Cleanup old backups beyond retention
    DELETE NOPROMPT OBSOLETE;

    -- Verify backup integrity
    CROSSCHECK BACKUP;
    CROSSCHECK ARCHIVELOG ALL;
}
```

### RMAN Recovery Scenarios

```sql
-- ═══════════════════════════════════════
-- SCENARIO 1: Complete Database Recovery (lost all datafiles)
-- ═══════════════════════════════════════

RMAN> STARTUP MOUNT;                    -- Can't OPEN (datafiles missing)
RMAN> RESTORE DATABASE;                 -- Restore from backup
RMAN> RECOVER DATABASE;                 -- Apply archive logs
RMAN> ALTER DATABASE OPEN;

-- ═══════════════════════════════════════
-- SCENARIO 2: Recover a single datafile
-- ═══════════════════════════════════════

-- Take datafile offline
SQL> ALTER DATABASE DATAFILE 7 OFFLINE;

RMAN> RESTORE DATAFILE 7;
RMAN> RECOVER DATAFILE 7;

SQL> ALTER DATABASE DATAFILE 7 ONLINE;

-- ═══════════════════════════════════════
-- SCENARIO 3: Point-in-Time Recovery (recover to before a bad DELETE)
-- ═══════════════════════════════════════

RMAN> STARTUP MOUNT;
RMAN> -- Restore to 5 minutes before the bad DELETE
RMAN> RESTORE DATABASE UNTIL TIME "TO_DATE('2026-06-02 14:25:00', 'YYYY-MM-DD HH24:MI:SS')";
RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2026-06-02 14:25:00', 'YYYY-MM-DD HH24:MI:SS')";
RMAN> ALTER DATABASE OPEN RESETLOGS;

-- ═══════════════════════════════════════
-- SCENARIO 4: Block-level recovery (single corrupt block)
-- ═══════════════════════════════════════

RMAN> -- Fix corrupt block in datafile 4, block 512
RMAN> BLOCKRECOVER DATAFILE 4 BLOCK 512;

-- ═══════════════════════════════════════
-- SCENARIO 5: Recover dropped tablespace
-- ═══════════════════════════════════════

-- Can't recover just a tablespace — must do full PITR
-- Use Flashback Database instead (if enabled) — MUCH faster!

-- ═══════════════════════════════════════
-- MONITORING BACKUPS
-- ═══════════════════════════════════════

RMAN> LIST BACKUP SUMMARY;

RMAN> LIST BACKUP OF DATABASE;

RMAN> REPORT NEED BACKUP;              -- What needs backing up?

RMAN> REPORT OBSOLETE;                 -- What can be deleted?

-- Check backup status from SQL
SELECT
    session_key, input_type, status,
    TO_CHAR(start_time, 'YYYY-MM-DD HH24:MI') AS start_time,
    TO_CHAR(end_time, 'YYYY-MM-DD HH24:MI')   AS end_time,
    elapsed_seconds / 60 AS elapsed_min,
    ROUND(input_bytes / 1024 / 1024 / 1024, 2) AS input_gb,
    ROUND(output_bytes / 1024 / 1024 / 1024, 2) AS output_gb,
    ROUND(compression_ratio, 1) AS compression
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 10 ROWS ONLY;
```

### RMAN Best Practices

```
╔══════════════════════════════════════════════════════════════════════╗
║  Practice                              │ Why                       ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  Enable Block Change Tracking          │ 10x faster incrementals   ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  Use compressed backups                │ 60-70% space savings      ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  Use an RMAN Catalog DB                │ Central backup metadata   ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  Parallelize with multiple channels    │ Faster backups            ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  Test restores REGULARLY               │ Backup is useless if     ║
║                                        │ restore doesn't work!    ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  Automate with DBMS_SCHEDULER          │ Never miss a backup      ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  VALIDATE BACKUPSET after backup       │ Catch corruption early    ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  DELETE OBSOLETE regularly             │ Don't fill up disk        ║
╠════════════════════════════════════════╪═══════════════════════════╣
║  Backup to tape AND disk (3-2-1 rule)  │ Disk = fast recovery      ║
║                                        │ Tape = disaster recovery  ║
╚════════════════════════════════════════╧═══════════════════════════╝
```

---

## 📦 Section 4: Data Pump — High-Speed Export/Import

### What is Data Pump?

**Data Pump** (expdp/impdp) is Oracle's modern export/import utility. It's orders of magnitude faster than the old exp/imp tools because it reads data directly from the datafiles (server-side) instead of going through SQL.

```
╔══════════════════════════════════════════════════════════════════╗
║          OLD exp/imp              DATA PUMP (expdp/impdp)       ║
║                                                                  ║
║   Client → SQL → Server        Server reads directly from       ║
║   → Format → Client            datafiles → dump file on SERVER  ║
║   → Write to client disk       (no network transfer!)           ║
║                                                                  ║
║   Speed: 😴 Slow               Speed: 🚀 10-50x faster         ║
║   Parallel: ❌ No              Parallel: ✅ Yes                 ║
║   Network: Heavy               Network: Zero (server-side)      ║
╚══════════════════════════════════════════════════════════════════╝
```

### Setting Up Data Pump

```sql
-- Create a directory object (Data Pump REQUIRES a server-side directory)
CREATE OR REPLACE DIRECTORY dp_dump_dir AS '/u01/exports';
GRANT READ, WRITE ON DIRECTORY dp_dump_dir TO app_user;

-- Verify
SELECT directory_name, directory_path FROM dba_directories;
```

### Data Pump Export (expdp)

```bash
# ═══════════════════════════════════════
# FULL DATABASE EXPORT
# ═══════════════════════════════════════
expdp system/password \
    FULL=Y \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=full_export_%U.dmp \
    LOGFILE=full_export.log \
    PARALLEL=4 \
    COMPRESSION=ALL \
    FILESIZE=10G

# ═══════════════════════════════════════
# SCHEMA EXPORT (most common use case ⭐)
# ═══════════════════════════════════════
expdp system/password \
    SCHEMAS=HR,FINANCE \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_finance_%U.dmp \
    LOGFILE=hr_finance_export.log \
    PARALLEL=4 \
    COMPRESSION=ALL

# ═══════════════════════════════════════
# TABLE EXPORT
# ═══════════════════════════════════════
expdp hr/password \
    TABLES=employees,departments \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_tables.dmp \
    LOGFILE=hr_tables_export.log \
    QUERY=employees:'"WHERE hire_date > DATE ''2025-01-01''"'

# ═══════════════════════════════════════
# EXPORT WITH DATA FILTERING
# ═══════════════════════════════════════
expdp system/password \
    SCHEMAS=HR \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_filtered.dmp \
    INCLUDE=TABLE:\"IN (\'EMPLOYEES\', \'DEPARTMENTS\')\" \
    CONTENT=DATA_ONLY        # DATA_ONLY | METADATA_ONLY | ALL

# ═══════════════════════════════════════
# EXPORT METADATA ONLY (schema structure without data)
# ═══════════════════════════════════════
expdp system/password \
    SCHEMAS=HR \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_metadata.dmp \
    CONTENT=METADATA_ONLY
```

### Data Pump Import (impdp)

```bash
# ═══════════════════════════════════════
# FULL IMPORT
# ═══════════════════════════════════════
impdp system/password \
    FULL=Y \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=full_export_%U.dmp \
    LOGFILE=full_import.log \
    PARALLEL=4

# ═══════════════════════════════════════
# SCHEMA IMPORT (into same schema name)
# ═══════════════════════════════════════
impdp system/password \
    SCHEMAS=HR \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_finance_%U.dmp \
    LOGFILE=hr_import.log \
    PARALLEL=4

# ═══════════════════════════════════════
# REMAP SCHEMA (import HR data into HR_DEV schema)
# ═══════════════════════════════════════
impdp system/password \
    SCHEMAS=HR \
    REMAP_SCHEMA=HR:HR_DEV \
    REMAP_TABLESPACE=USERS:DEV_DATA \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_finance_%U.dmp \
    LOGFILE=hr_remap_import.log

# ═══════════════════════════════════════
# TABLE IMPORT WITH TABLE RENAME
# ═══════════════════════════════════════
impdp system/password \
    TABLES=HR.EMPLOYEES \
    REMAP_TABLE=HR.EMPLOYEES:EMPLOYEES_ARCHIVE \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_tables.dmp

# ═══════════════════════════════════════
# IMPORT WITH TABLE_EXISTS_ACTION
# ═══════════════════════════════════════
impdp system/password \
    SCHEMAS=HR \
    DIRECTORY=dp_dump_dir \
    DUMPFILE=hr_finance_%U.dmp \
    TABLE_EXISTS_ACTION=TRUNCATE    # SKIP | APPEND | TRUNCATE | REPLACE
```

### Data Pump — Monitoring & Management

```sql
-- Monitor running Data Pump jobs
SELECT
    owner_name, job_name, operation, job_mode,
    state, degree,
    attached_sessions
FROM dba_datapump_jobs
WHERE state = 'EXECUTING';

-- Attach to a running job (to monitor or control it)
-- expdp system/password ATTACH=SYS_EXPORT_SCHEMA_01

-- Inside attached session:
-- Export> STATUS          -- Show progress
-- Export> PARALLEL=8      -- Increase parallelism on the fly
-- Export> KILL_JOB        -- Cancel the job
-- Export> STOP_JOB        -- Pause the job
-- Export> START_JOB       -- Resume the job
```

---

## 🔧 Section 5: Oracle Patching

### Why Patching Matters

```
╔══════════════════════════════════════════════════════════════════╗
║              ORACLE PATCH TYPES                                  ║
║                                                                  ║
║   ┌─────────────────────────────────────────────────────┐       ║
║   │  Release Update (RU)                                │       ║
║   │  → Quarterly cumulative patch (Jan, Apr, Jul, Oct) │       ║
║   │  → Contains bug fixes + security fixes              │       ║
║   │  → MOST IMPORTANT — apply every quarter ⭐         │       ║
║   └─────────────────────────────────────────────────────┘       ║
║                                                                  ║
║   ┌─────────────────────────────────────────────────────┐       ║
║   │  Release Update Revision (RUR)                      │       ║
║   │  → Like RU but with additional regression fixes     │       ║
║   │  → For production systems that need extra stability │       ║
║   └─────────────────────────────────────────────────────┘       ║
║                                                                  ║
║   ┌─────────────────────────────────────────────────────┐       ║
║   │  One-off Patch                                      │       ║
║   │  → Single bug fix, usually from Oracle Support      │       ║
║   │  → Applied between quarterly RUs                    │       ║
║   └─────────────────────────────────────────────────────┘       ║
║                                                                  ║
║   ⚠️ Unpatched databases = SECURITY VULNERABILITIES             ║
║   Oracle releases Critical Patch Updates (CPU) quarterly        ║
╚══════════════════════════════════════════════════════════════════╝
```

### Patching Workflow

```
╔══════════════════════════════════════════════════════════════════════╗
║              ORACLE PATCHING WORKFLOW                                ║
║                                                                      ║
║   Step 1: PLAN                                                      ║
║   │  → Download patch from My Oracle Support (MOS)                  ║
║   │  → Read the README (patch conflicts, prerequisites)             ║
║   │  → Schedule downtime window                                     ║
║   │                                                                  ║
║   Step 2: PRE-PATCH                                                 ║
║   │  → RMAN backup (FULL!) — safety net                            ║
║   │  → Check current patch level: opatch lsinventory                ║
║   │  → Verify OPatch version (upgrade if needed)                    ║
║   │  → Run conflict detection: opatch prereq CheckConflictAgainstOH ║
║   │                                                                  ║
║   Step 3: APPLY                                                     ║
║   │  → Shutdown database                                            ║
║   │  → Unzip patch to staging area                                  ║
║   │  → cd $ORACLE_HOME && opatch apply /path/to/patch               ║
║   │  → Start database                                               ║
║   │  → Run datapatch (applies SQL changes)                          ║
║   │                                                                  ║
║   Step 4: POST-PATCH                                                ║
║   │  → Verify: opatch lsinventory                                   ║
║   │  → Run utlrp.sql (recompile invalid objects)                    ║
║   │  → Test application functionality                               ║
║   │  → Monitor alert log for errors                                  ║
║   │                                                                  ║
║   Step 5: ROLLBACK (if something goes wrong)                        ║
║       → opatch rollback -id <patch_id>                              ║
║       → Or restore from RMAN backup (worst case)                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Patching Commands

```bash
# Check current patch level
$ORACLE_HOME/OPatch/opatch lsinventory

# Check OPatch version
$ORACLE_HOME/OPatch/opatch version

# Apply a patch
cd /tmp/patch/12345678
$ORACLE_HOME/OPatch/opatch apply

# Apply Release Update (RU) — typically larger
cd /tmp/patch/36xxxxx
$ORACLE_HOME/OPatch/opatch apply

# After starting the database, run datapatch
$ORACLE_HOME/OPatch/datapatch -verbose

# Recompile invalid objects
sqlplus / as sysdba
@?/rdbms/admin/utlrp.sql

# Check for invalid objects
SELECT owner, object_name, object_type, status
FROM dba_objects
WHERE status = 'INVALID'
ORDER BY owner, object_type, object_name;

# Rollback a patch
$ORACLE_HOME/OPatch/opatch rollback -id 12345678
```

### Out-of-Place Patching (Recommended for RAC)

```
╔══════════════════════════════════════════════════════════════════╗
║           OUT-OF-PLACE PATCHING (Zero Downtime with RAC)        ║
║                                                                  ║
║   Instead of patching the running ORACLE_HOME:                  ║
║                                                                  ║
║   1. Clone ORACLE_HOME to a NEW location                        ║
║   2. Apply patch to the NEW home (while DB runs on OLD home)    ║
║   3. Switch each RAC node to the NEW home (rolling)             ║
║   4. Result: ZERO downtime for the application!                 ║
║                                                                  ║
║   Node 1: OLD_HOME (running) → shut down → switch to NEW_HOME  ║
║   Node 2: OLD_HOME (still running, serving users)              ║
║   Node 1: NEW_HOME (started, serving users)                    ║
║   Node 2: OLD_HOME → shut down → switch to NEW_HOME            ║
║   Node 2: NEW_HOME (started, serving users)                    ║
║                                                                  ║
║   Total application downtime: ZERO 🎯                           ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 📋 Section 6: Daily DBA Monitoring Queries

### The DBA Dashboard — Queries You Run Every Day

```sql
-- 1️⃣  Alert Log — Check for ORA- errors (CRITICAL ⭐)
SELECT originating_timestamp, message_text
FROM v$diag_alert_ext
WHERE originating_timestamp > SYSDATE - 1
  AND message_text LIKE '%ORA-%'
ORDER BY originating_timestamp DESC;

-- 2️⃣  Tablespace Usage (run daily)
-- (Query from Section 1 above)

-- 3️⃣  Long Running Sessions
SELECT
    s.sid, s.serial#, s.username, s.program, s.module,
    s.sql_id, s.status,
    ROUND(s.last_call_et / 3600, 1) AS hours_active,
    s.blocking_session
FROM v$session s
WHERE s.type = 'USER'
  AND s.status = 'ACTIVE'
  AND s.last_call_et > 3600  -- Running for more than 1 hour
ORDER BY s.last_call_et DESC;

-- 4️⃣  Blocking Sessions
SELECT
    s1.sid AS blocker_sid,
    s1.username AS blocker_user,
    s1.sql_id AS blocker_sql,
    s2.sid AS waiter_sid,
    s2.username AS waiter_user,
    s2.sql_id AS waiter_sql,
    s2.seconds_in_wait
FROM v$session s1
JOIN v$session s2 ON s1.sid = s2.blocking_session
ORDER BY s2.seconds_in_wait DESC;

-- 5️⃣  Database Size
SELECT
    ROUND(SUM(bytes) / 1024 / 1024 / 1024, 2) AS db_size_gb
FROM dba_data_files;

-- 6️⃣  Backup Status (last 7 days)
SELECT
    input_type, status,
    TO_CHAR(start_time, 'YYYY-MM-DD HH24:MI') AS start_time,
    ROUND(elapsed_seconds / 60) AS elapsed_min,
    ROUND(output_bytes / 1024 / 1024 / 1024, 2) AS output_gb
FROM v$rman_backup_job_details
WHERE start_time > SYSDATE - 7
ORDER BY start_time DESC;

-- 7️⃣  Invalid Objects
SELECT owner, object_type, COUNT(*) AS invalid_count
FROM dba_objects
WHERE status = 'INVALID'
GROUP BY owner, object_type
ORDER BY owner, object_type;

-- 8️⃣  Jobs/Scheduler Status
SELECT
    owner, job_name, state, last_start_date,
    next_run_date, run_count, failure_count
FROM dba_scheduler_jobs
WHERE enabled = 'TRUE'
ORDER BY next_run_date;
```

---

## 🧪 Interview-Ready Explanations

### "Walk me through an Oracle backup strategy"

> *"I use RMAN with an incremental strategy: Level 0 full backup on Sundays, Level 1 cumulative incrementals Monday through Saturday, and archive log backups every hour. Block change tracking is enabled for faster incrementals. Backups are compressed and parallelized across 4 channels. I maintain a 7-day recovery window and test restores monthly. For disaster recovery, backup sets are also copied to a remote site."*

### Quick-Fire Interview Questions

| Question | Answer |
|----------|--------|
| "What's a tablespace?" | Logical storage container. Tables and indexes live inside tablespaces. A tablespace has one or more physical datafiles on disk. |
| "RMAN vs Data Pump?" | RMAN = backup/recovery (bit-level, point-in-time). Data Pump = logical export/import (schema/table level, for migration/refresh). |
| "Level 0 vs Level 1?" | Level 0 = full baseline backup. Level 1 = only blocks changed since last Level 0 (or Level 1). |
| "expdp vs exp?" | expdp (Data Pump) = server-side, parallel, 10-50x faster. exp (legacy) = client-side, single-threaded, deprecated. |
| "How to handle ORA-01555?" | Undo tablespace too small or undo retention too short. Increase UNDO_RETENTION and UNDO tablespace size. |
| "What is ASM?" | Automatic Storage Management — Oracle's built-in volume manager/filesystem. Manages disks, striping, and mirroring for Oracle. |
| "Roles vs Privileges?" | Privileges = atomic permissions (SELECT, CREATE TABLE). Roles = bundles of privileges for easier management. |

---

## 🔑 Key Takeaways

```
✅ Tablespaces = logical containers → Datafiles = physical storage on disk
✅ NEVER put user data in SYSTEM or SYSAUX tablespaces
✅ Use custom ROLES instead of granting privileges directly to users
✅ RMAN is THE backup tool — use incremental + compressed + parallel
✅ Level 0 weekly + Level 1 daily + archive logs hourly = solid strategy
✅ TEST your restores! A backup you can't restore is worthless
✅ Data Pump (expdp/impdp) for schema migration, dev refresh, data transfer
✅ REMAP_SCHEMA in impdp is a lifesaver for dev/test environment refresh
✅ Patch QUARTERLY (Release Updates) — unpatched = vulnerable
✅ Monitor daily: alert log, tablespace space, backups, blocking sessions
✅ ASM simplifies storage — use it in production environments
```

---

## 🔗 What's Next?

**Chapter 2B.8 → [Oracle Multitenant & Pluggable Databases](./08-Oracle-Multitenant.md)**
Where we learn Oracle's **game-changing architecture** — one container, many databases, ultimate consolidation.

---

> *"A DBA who doesn't test restores is just collecting backup files for fun."* — Every Senior DBA Ever
