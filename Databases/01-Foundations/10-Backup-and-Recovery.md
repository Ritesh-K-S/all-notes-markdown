# 💾 Chapter 1.10 — Backup, Recovery & Disaster Planning

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 1.3 (DBMS Architecture), Chapter 1.9 (Database Security)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **Full, Incremental, and Differential** backup strategies
- Know how **Point-in-Time Recovery (PITR)** works across all major databases
- Design a **disaster recovery plan** with proper RTO/RPO
- Implement backup strategies for **Oracle, SQL Server, PostgreSQL, MySQL, and MongoDB**
- Know the difference between **logical and physical** backups — and when to use each

---

## 🧠 Why Backups Are Non-Negotiable — Horror Stories

```
╔══════════════════════════════════════════════════════════════════╗
║  REAL DISASTERS THAT HAPPENED:                                   ║
║                                                                  ║
║  🔥 GitLab (2017): Accidentally deleted production DB.          ║
║     5 backup methods configured. NONE worked properly.          ║
║     Lost 6 hours of data. Public post-mortem.                   ║
║                                                                  ║
║  🔥 Toy Story 2 (1998): Someone ran 'rm -rf *' on the          ║
║     animation files. Backups? Corrupted.                        ║
║     Saved only because an employee had a copy at home.          ║
║                                                                  ║
║  🔥 Ma.gnolia (2009): Social bookmarking site.                  ║
║     Database corruption + no working backup = ALL user data     ║
║     permanently lost. Company shut down.                        ║
║                                                                  ║
║  RULE #1: A backup you haven't tested is NOT a backup.          ║
║  RULE #2: If your data exists in only one place, it doesn't     ║
║           exist.                                                 ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 📐 Key Concepts — RTO and RPO

```
                         DISASTER TIMELINE

  Last          Disaster      Detection     Recovery
  Backup        Strikes       & Response    Complete
    │              │              │             │
    ▼              ▼              ▼             ▼
────┼──────────────┼──────────────┼─────────────┼────────►
    │              │              │             │        Time
    │◄────────────►│              │◄───────────►│
    │     RPO      │              │     RTO     │
    │  (Data you   │              │  (Downtime  │
    │   can lose)  │              │   allowed)  │
    │              │              │             │


  ╔══════════════════════════════════════════════════════════════╗
  ║  RPO — Recovery Point Objective                              ║
  ║  "How much data can we AFFORD TO LOSE?"                      ║
  ║                                                              ║
  ║  RPO = 0       → Zero data loss (synchronous replication)    ║
  ║  RPO = 1 hour  → Can lose up to 1 hour of data              ║
  ║  RPO = 24 hours→ Daily backups are sufficient                ║
  ╠══════════════════════════════════════════════════════════════╣
  ║  RTO — Recovery Time Objective                               ║
  ║  "How long can we be DOWN?"                                  ║
  ║                                                              ║
  ║  RTO = 0       → Zero downtime (hot standby / failover)     ║
  ║  RTO = 1 hour  → Must be back online within 1 hour          ║
  ║  RTO = 24 hours→ Can take a day to restore                  ║
  ╚══════════════════════════════════════════════════════════════╝

  ┌───────────────────┬─────────┬─────────┬─────────────────────┐
  │ System Type       │ RPO     │ RTO     │ Strategy            │
  ├───────────────────┼─────────┼─────────┼─────────────────────┤
  │ Banking / Trading │ 0       │ Minutes │ Sync replication +  │
  │                   │         │         │ hot standby         │
  │ E-Commerce        │ Minutes │ < 1 hr  │ Async replication + │
  │                   │         │         │ warm standby        │
  │ Internal Apps     │ Hours   │ < 4 hrs │ Frequent backups +  │
  │                   │         │         │ restore procedures  │
  │ Dev / Staging     │ Days    │ Days    │ Daily backups       │
  │ Archive / Audit   │ N/A     │ Days    │ Weekly + tape       │
  └───────────────────┴─────────┴─────────┴─────────────────────┘
```

---

## 📦 Backup Types — The Three Strategies

### 1. Full Backup

> Copy **EVERYTHING**. The simplest and most reliable, but slowest and largest.

```
Day 1 (Monday):     FULL BACKUP ████████████████████████ (100 GB)
Day 2 (Tuesday):    FULL BACKUP ████████████████████████ (102 GB)
Day 3 (Wednesday):  FULL BACKUP ████████████████████████ (105 GB)
Day 4 (Thursday):   FULL BACKUP ████████████████████████ (107 GB)

Storage used: 100 + 102 + 105 + 107 = 414 GB 💸
Restore time: Fast (single file to restore)
Risk:         Low (each backup is independent)
```

### 2. Differential Backup

> Copy everything that **changed since the LAST FULL backup**.

```
Day 1 (Monday):     FULL BACKUP ████████████████████████ (100 GB)
Day 2 (Tuesday):    DIFF BACKUP ███                      (5 GB)   ← changes since Mon
Day 3 (Wednesday):  DIFF BACKUP █████                    (10 GB)  ← changes since Mon
Day 4 (Thursday):   DIFF BACKUP ████████                 (17 GB)  ← changes since Mon
                                                          ↑ Growing larger each day!

Storage used: 100 + 5 + 10 + 17 = 132 GB ✅ Much less!

To restore Wednesday's data:
  Step 1: Restore Monday's FULL backup
  Step 2: Apply Wednesday's DIFFERENTIAL
  Done! (Only 2 files needed)
```

### 3. Incremental Backup

> Copy everything that **changed since the LAST BACKUP** (full OR incremental).

```
Day 1 (Monday):     FULL BACKUP ████████████████████████ (100 GB)
Day 2 (Tuesday):    INCR BACKUP ██                       (3 GB)   ← changes since Mon
Day 3 (Wednesday):  INCR BACKUP █                        (2 GB)   ← changes since Tue
Day 4 (Thursday):   INCR BACKUP ██                       (4 GB)   ← changes since Wed

Storage used: 100 + 3 + 2 + 4 = 109 GB ✅ Smallest!

To restore Thursday's data:
  Step 1: Restore Monday's FULL backup
  Step 2: Apply Tuesday's INCREMENTAL
  Step 3: Apply Wednesday's INCREMENTAL
  Step 4: Apply Thursday's INCREMENTAL
  Done! (4 files needed — slower restore, but smallest backups)
```

### Comparison

```
╔═══════════════════╦═══════════════╦════════════════╦═════════════════╗
║                   ║ Full          ║ Differential   ║ Incremental     ║
╠═══════════════════╬═══════════════╬════════════════╬═════════════════╣
║ Backup speed      ║ 🐌 Slowest    ║ 🟡 Medium      ║ ⚡ Fastest       ║
║ Backup size       ║ 💾 Largest    ║ 💾 Medium      ║ 💾 Smallest     ║
║ Restore speed     ║ ⚡ Fastest    ║ 🟡 Medium      ║ 🐌 Slowest      ║
║ Restore steps     ║ 1 file       ║ 2 files        ║ Many files      ║
║ Risk if one       ║ Only that    ║ Full + that    ║ Chain breaks =  ║
║ backup corrupts   ║ day lost     ║ diff lost      ║ all after lost! ║
║ Storage cost      ║ 💰💰💰       ║ 💰💰           ║ 💰              ║
╚═══════════════════╩═══════════════╩════════════════╩═════════════════╝

  💡 COMMON STRATEGY (Best of all worlds):
  
  Weekly Full + Daily Differential + Hourly Transaction Log
  
  Sunday:    FULL ████████████████████████
  Monday:    DIFF ██
  Tuesday:   DIFF ████
  Wednesday: DIFF ██████
  Thursday:  DIFF ████████
  Friday:    DIFF ██████████
  Saturday:  DIFF ████████████
  Sunday:    FULL ████████████████████████  (new cycle)
  
  + Transaction log backups every 15 minutes (for PITR)
```

---

## 🔄 Physical vs Logical Backups

```
╔══════════════════════════════════════════════════════════════════════╗
║              PHYSICAL BACKUP                LOGICAL BACKUP          ║
╠══════════════════════════════════════════════════════════════════════╣
║ Copies raw data files                 Exports data as SQL/JSON/    ║
║ (binary, exact copy)                  CSV (human-readable)          ║
║                                                                      ║
║ ┌──────────────────┐                 ┌──────────────────┐           ║
║ │ data files       │                 │ CREATE TABLE...  │           ║
║ │ WAL/redo logs    │                 │ INSERT INTO...   │           ║
║ │ config files     │                 │ INSERT INTO...   │           ║
║ │ (binary format)  │                 │ (text format)    │           ║
║ └──────────────────┘                 └──────────────────┘           ║
║                                                                      ║
║ ✅ Fast backup & restore             ✅ Portable (cross-version,    ║
║ ✅ Supports PITR                        cross-platform)             ║
║ ✅ Consistent state                  ✅ Selective (specific tables) ║
║                                      ✅ Human-readable              ║
║ ❌ Same DB version required          ❌ Slower backup & restore     ║
║ ❌ Same platform required            ❌ No PITR support             ║
║ ❌ Not human-readable                ❌ May miss some metadata      ║
║                                                                      ║
║ TOOLS:                               TOOLS:                         ║
║ • Oracle: RMAN                       • Oracle: expdp/impdp          ║
║ • SQL Server: Native backup          • SQL Server: BCP, BACPAC     ║
║ • PostgreSQL: pg_basebackup          • PostgreSQL: pg_dump          ║
║ • MySQL: Percona XtraBackup          • MySQL: mysqldump             ║
║ • MongoDB: filesystem snapshot       • MongoDB: mongodump           ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🕐 Point-in-Time Recovery (PITR) — The Magic

> **PITR lets you restore your database to ANY specific moment in time** — not just the last backup.

### How PITR Works

```
  Full Backup            Changes are logged continuously
  (Sunday 2AM)           in transaction/WAL/redo logs
       │                        │
       ▼                        ▼
  ┌─────────┐  ┌─────┬─────┬─────┬─────┬─────┬─────┐
  │ FULL    │  │ Log │ Log │ Log │ Log │ Log │ Log │
  │ BACKUP  │  │ 8AM │ 9AM │10AM │11AM │12PM │ 1PM │
  │ Sun 2AM │  │ Mon │ Mon │ Mon │ Mon │ Mon │ Mon │
  └─────────┘  └─────┴─────┴─────┴─────┴─────┴─────┘
       │                              ↑
       │                              │
       │    "Restore to Monday 10:35 AM"
       │                              │
       └──► Apply FULL backup         │
            Apply logs up to ──────►──┘
            Mon 10:35 AM
            
  Result: Database state at EXACTLY Monday 10:35:00 AM! 🎯

  Use case: "Someone deleted the users table at 10:37 AM.
             Restore to 10:35 AM — just before the disaster!"
```

---

## 🛠️ Backup Implementation — Every Major Database

### PostgreSQL

```bash
# ═══════════════════════════════════════════
# LOGICAL BACKUP — pg_dump
# ═══════════════════════════════════════════

# Dump entire database (SQL format)
pg_dump -h localhost -U postgres -d myapp -F p -f backup.sql

# Dump in custom format (compressed, parallel restore)
pg_dump -h localhost -U postgres -d myapp -F c -f backup.dump

# Dump specific tables only
pg_dump -h localhost -U postgres -d myapp -t orders -t users -F c -f partial.dump

# Dump ALL databases
pg_dumpall -h localhost -U postgres -f all_databases.sql

# Restore from logical backup
psql -h localhost -U postgres -d myapp -f backup.sql              # SQL format
pg_restore -h localhost -U postgres -d myapp -j 4 backup.dump     # Custom format, 4 parallel jobs

# ═══════════════════════════════════════════
# PHYSICAL BACKUP — pg_basebackup (supports PITR)
# ═══════════════════════════════════════════

# Prerequisites: postgresql.conf
#   wal_level = replica
#   archive_mode = on
#   archive_command = 'cp %p /backup/wal_archive/%f'

# Take a base backup
pg_basebackup -h localhost -U replication_user \
    -D /backup/base/2024-03-15 \
    -Ft -z -P                        # tar format, compressed, progress

# ═══════════════════════════════════════════
# POINT-IN-TIME RECOVERY
# ═══════════════════════════════════════════

# Step 1: Stop PostgreSQL
# Step 2: Clear the data directory
# Step 3: Restore base backup to data directory
# Step 4: Create recovery.signal file
# Step 5: Configure postgresql.conf:
#   restore_command = 'cp /backup/wal_archive/%f %p'
#   recovery_target_time = '2024-03-15 10:35:00'
#   recovery_target_action = 'promote'
# Step 6: Start PostgreSQL — it replays WAL up to the target time!
```

### MySQL

```bash
# ═══════════════════════════════════════════
# LOGICAL BACKUP — mysqldump
# ═══════════════════════════════════════════

# Dump entire database
mysqldump -u root -p --single-transaction --routines --triggers \
    myapp > backup.sql

# Dump with master position (for replication setup)
mysqldump -u root -p --single-transaction --master-data=2 \
    --all-databases > full_backup.sql

# Dump specific tables
mysqldump -u root -p myapp orders users > tables_backup.sql

# Restore
mysql -u root -p myapp < backup.sql

# ═══════════════════════════════════════════
# PHYSICAL BACKUP — Percona XtraBackup (recommended)
# ═══════════════════════════════════════════

# Full backup
xtrabackup --backup --target-dir=/backup/full/ \
    --user=root --password=xxx

# Incremental backup (since last full)
xtrabackup --backup --target-dir=/backup/inc1/ \
    --incremental-basedir=/backup/full/ \
    --user=root --password=xxx

# Prepare and restore
xtrabackup --prepare --target-dir=/backup/full/
xtrabackup --prepare --target-dir=/backup/full/ \
    --incremental-dir=/backup/inc1/
xtrabackup --copy-back --target-dir=/backup/full/

# ═══════════════════════════════════════════
# POINT-IN-TIME RECOVERY (using binary logs)
# ═══════════════════════════════════════════

# Prerequisites: my.cnf
#   log-bin = mysql-bin
#   binlog_format = ROW
#   expire_logs_days = 14

# Step 1: Restore the last full backup
mysql -u root -p < full_backup.sql

# Step 2: Apply binary logs up to the target time
mysqlbinlog --stop-datetime="2024-03-15 10:35:00" \
    /var/lib/mysql/mysql-bin.000042 \
    /var/lib/mysql/mysql-bin.000043 | mysql -u root -p
```

### SQL Server

```sql
-- ═══════════════════════════════════════════
-- FULL BACKUP
-- ═══════════════════════════════════════════
BACKUP DATABASE MyApp
    TO DISK = 'D:\Backups\MyApp_Full_20240315.bak'
    WITH COMPRESSION, CHECKSUM, STATS = 10;

-- ═══════════════════════════════════════════
-- DIFFERENTIAL BACKUP
-- ═══════════════════════════════════════════
BACKUP DATABASE MyApp
    TO DISK = 'D:\Backups\MyApp_Diff_20240315.bak'
    WITH DIFFERENTIAL, COMPRESSION, CHECKSUM;

-- ═══════════════════════════════════════════
-- TRANSACTION LOG BACKUP (every 15 minutes for PITR)
-- ═══════════════════════════════════════════
BACKUP LOG MyApp
    TO DISK = 'D:\Backups\MyApp_Log_20240315_1035.trn'
    WITH COMPRESSION, CHECKSUM;

-- ═══════════════════════════════════════════
-- RESTORE (Full + Diff + Logs to a point in time)
-- ═══════════════════════════════════════════

-- Step 1: Restore FULL backup (with NORECOVERY — more files coming)
RESTORE DATABASE MyApp
    FROM DISK = 'D:\Backups\MyApp_Full_20240310.bak'
    WITH NORECOVERY, REPLACE;

-- Step 2: Restore DIFFERENTIAL (with NORECOVERY — logs still coming)
RESTORE DATABASE MyApp
    FROM DISK = 'D:\Backups\MyApp_Diff_20240315.bak'
    WITH NORECOVERY;

-- Step 3: Restore transaction logs up to target time
RESTORE LOG MyApp
    FROM DISK = 'D:\Backups\MyApp_Log_20240315_1030.trn'
    WITH NORECOVERY;

RESTORE LOG MyApp
    FROM DISK = 'D:\Backups\MyApp_Log_20240315_1035.trn'
    WITH STOPAT = '2024-03-15T10:35:00',    -- Stop at exact time!
         RECOVERY;                            -- Make DB available

-- ═══════════════════════════════════════════
-- AUTOMATED BACKUP with SQL Agent Job
-- ═══════════════════════════════════════════
-- Use Maintenance Plans in SSMS or Ola Hallengren's scripts:
-- https://ola.hallengren.com (industry standard)
```

### Oracle

```bash
# ═══════════════════════════════════════════
# RMAN — Recovery Manager (THE standard for Oracle backups)
# ═══════════════════════════════════════════

# Connect to RMAN
rman target /

# Full backup (compressed)
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE
      PLUS ARCHIVELOG;

# Incremental Level 0 (base for incrementals)
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;

# Incremental Level 1 (changes since last level 0 or level 1)
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;

# Block Change Tracking (speeds up incrementals dramatically)
RMAN> ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
      USING FILE '/oracle/bct/bct_file.ctf';

# ═══════════════════════════════════════════
# POINT-IN-TIME RECOVERY with RMAN
# ═══════════════════════════════════════════

RMAN> RUN {
        SET UNTIL TIME "TO_DATE('2024-03-15 10:35:00', 'YYYY-MM-DD HH24:MI:SS')";
        RESTORE DATABASE;
        RECOVER DATABASE;
      }

RMAN> ALTER DATABASE OPEN RESETLOGS;

# ═══════════════════════════════════════════
# Data Pump — Logical Backup (Oracle)
# ═══════════════════════════════════════════

# Export entire schema
expdp system/password SCHEMAS=myapp DIRECTORY=backup_dir \
    DUMPFILE=myapp_20240315.dmp LOGFILE=export.log

# Export specific tables with WHERE clause
expdp system/password TABLES=myapp.orders \
    QUERY=\"WHERE order_date > DATE '2024-01-01'\" \
    DIRECTORY=backup_dir DUMPFILE=orders.dmp

# Import
impdp system/password SCHEMAS=myapp DIRECTORY=backup_dir \
    DUMPFILE=myapp_20240315.dmp LOGFILE=import.log
```

### MongoDB

```bash
# ═══════════════════════════════════════════
# LOGICAL BACKUP — mongodump
# ═══════════════════════════════════════════

# Dump entire database
mongodump --uri="mongodb://user:pass@host:27017/myapp" \
    --out=/backup/2024-03-15/

# Dump specific collection
mongodump --uri="mongodb://user:pass@host:27017/myapp" \
    --collection=orders --out=/backup/orders/

# Dump with query (only recent data)
mongodump --uri="mongodb://user:pass@host:27017/myapp" \
    --collection=orders \
    --query='{"createdAt": {"$gte": {"$date": "2024-01-01T00:00:00Z"}}}'

# Restore
mongorestore --uri="mongodb://user:pass@host:27017/myapp" \
    /backup/2024-03-15/myapp/

# ═══════════════════════════════════════════
# POINT-IN-TIME RECOVERY (Replica Set + Oplog)
# ═══════════════════════════════════════════

# Dump with oplog (captures operations during backup)
mongodump --uri="mongodb://host:27017" --oplog --out=/backup/full/

# Restore to specific point in time
mongorestore --uri="mongodb://host:27017" \
    --oplogReplay \
    --oplogLimit="1710500100:1" \   # Timestamp to stop at
    /backup/full/

# ═══════════════════════════════════════════
# MongoDB Atlas — Automated Backups
# ═══════════════════════════════════════════
# Atlas provides:
# • Continuous backups with PITR (last 7 days)
# • Snapshot backups (configurable retention)
# • One-click restore in the UI
# • Cross-region restore for DR
```

---

## 🏗️ Disaster Recovery Strategies

```
╔══════════════════════════════════════════════════════════════════════╗
║                    DISASTER RECOVERY TIERS                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  TIER 1: Cold Standby (Cheapest, Slowest)                           ║
║  ┌─────────────────────────────────────────┐                        ║
║  │ Primary DB ──backup──► Storage (S3/NFS) │                        ║
║  │                                         │                        ║
║  │ On disaster: Provision new server →     │                        ║
║  │              Restore from backup →      │                        ║
║  │              Redirect traffic            │                        ║
║  │ RTO: Hours | RPO: Last backup           │                        ║
║  └─────────────────────────────────────────┘                        ║
║                                                                      ║
║  TIER 2: Warm Standby (Balanced)                                    ║
║  ┌─────────────────────────────────────────┐                        ║
║  │ Primary DB ──async replicate──► Standby │                        ║
║  │     ✅                    (lag: minutes) │                        ║
║  │                                         │                        ║
║  │ On disaster: Promote standby →          │                        ║
║  │              Redirect traffic            │                        ║
║  │ RTO: Minutes | RPO: Minutes             │                        ║
║  └─────────────────────────────────────────┘                        ║
║                                                                      ║
║  TIER 3: Hot Standby (Most expensive, fastest)                      ║
║  ┌─────────────────────────────────────────┐                        ║
║  │ Primary DB ──sync replicate──► Standby  │                        ║
║  │     ✅              (lag: 0)     ✅      │                        ║
║  │                                         │                        ║
║  │ On disaster: Automatic failover →       │                        ║
║  │              Zero manual intervention    │                        ║
║  │ RTO: Seconds | RPO: 0                   │                        ║
║  └─────────────────────────────────────────┘                        ║
║                                                                      ║
║  TIER 4: Multi-Region Active-Active (Enterprise)                    ║
║  ┌─────────────────────────────────────────┐                        ║
║  │  US-East ◄──sync──► EU-West             │                        ║
║  │    ✅         ▲         ✅               │                        ║
║  │               │                          │                        ║
║  │           APAC-South                     │                        ║
║  │              ✅                          │                        ║
║  │ ALL regions serve traffic simultaneously │                        ║
║  │ RTO: 0 | RPO: 0 | Cost: 💰💰💰💰        │                        ║
║  └─────────────────────────────────────────┘                        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### DR by Database Technology

```
┌─────────────┬─────────────────────────────────────────────────────┐
│ Database    │ High Availability / DR Options                      │
├─────────────┼─────────────────────────────────────────────────────┤
│ PostgreSQL  │ • Streaming Replication (async/sync)                │
│             │ • Patroni (automatic failover)                      │
│             │ • PgBouncer (connection pooling)                    │
│             │ • pg_basebackup + WAL archiving (PITR)              │
├─────────────┼─────────────────────────────────────────────────────┤
│ MySQL       │ • Group Replication / InnoDB Cluster                │
│             │ • MySQL Router (proxy/failover)                     │
│             │ • Master-Slave replication                          │
│             │ • Percona XtraBackup (PITR)                         │
├─────────────┼─────────────────────────────────────────────────────┤
│ SQL Server  │ • Always On Availability Groups                     │
│             │ • Failover Cluster Instances (FCI)                  │
│             │ • Log Shipping                                      │
│             │ • Azure SQL Auto-failover groups                    │
├─────────────┼─────────────────────────────────────────────────────┤
│ Oracle      │ • Data Guard (Active standby)                       │
│             │ • Real Application Clusters (RAC)                   │
│             │ • GoldenGate (multi-master replication)             │
│             │ • RMAN + Archive logs (PITR)                        │
├─────────────┼─────────────────────────────────────────────────────┤
│ MongoDB     │ • Replica Sets (automatic failover)                 │
│             │ • Atlas Global Clusters (multi-region)              │
│             │ • Continuous backups + PITR (Atlas)                 │
├─────────────┼─────────────────────────────────────────────────────┤
│ Redis       │ • Redis Sentinel (automatic failover)               │
│             │ • Redis Cluster (sharding + replication)            │
│             │ • RDB snapshots + AOF persistence                  │
├─────────────┼─────────────────────────────────────────────────────┤
│ Cassandra   │ • Built-in: Multi-DC replication                    │
│             │ • No single point of failure (peer-to-peer)        │
│             │ • Snapshot + incremental backup                     │
└─────────────┴─────────────────────────────────────────────────────┘
```

---

## 📋 Backup Strategy Template

```
╔══════════════════════════════════════════════════════════════════╗
║             PRODUCTION BACKUP STRATEGY TEMPLATE                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DATABASE: ________________     SIZE: _______ GB                 ║
║  RPO: ___________              RTO: ___________                  ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────┐        ║
║  │ SCHEDULE                                            │        ║
║  ├─────────────────────────────────────────────────────┤        ║
║  │ Full Backup:     Weekly (Sunday 2:00 AM)            │        ║
║  │ Differential:    Daily (2:00 AM, Mon-Sat)           │        ║
║  │ Transaction Log: Every 15 minutes                    │        ║
║  └─────────────────────────────────────────────────────┘        ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────┐        ║
║  │ RETENTION                                           │        ║
║  ├─────────────────────────────────────────────────────┤        ║
║  │ Full Backups:    Keep for 4 weeks                   │        ║
║  │ Differentials:   Keep for 1 week                    │        ║
║  │ Transaction Logs: Keep for 7 days                    │        ║
║  │ Monthly Fulls:   Keep for 1 year (compliance)        │        ║
║  │ Yearly Fulls:    Keep for 7 years (legal)            │        ║
║  └─────────────────────────────────────────────────────┘        ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────┐        ║
║  │ STORAGE                                             │        ║
║  ├─────────────────────────────────────────────────────┤        ║
║  │ Primary:  Local SSD (fast restore)                  │        ║
║  │ Secondary: Network storage / NFS                    │        ║
║  │ Off-site:  S3 / Azure Blob / GCS (different region) │        ║
║  │ Encryption: AES-256 for all backup files            │        ║
║  └─────────────────────────────────────────────────────┘        ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────┐        ║
║  │ TESTING                                             │        ║
║  ├─────────────────────────────────────────────────────┤        ║
║  │ ✅ Restore test: Monthly (to staging environment)   │        ║
║  │ ✅ PITR test: Quarterly                             │        ║
║  │ ✅ DR failover test: Twice a year                   │        ║
║  │ ✅ Backup integrity: CHECKSUM on every backup       │        ║
║  │ ✅ Alert on backup failure: PagerDuty / email       │        ║
║  └─────────────────────────────────────────────────────┘        ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## ⚠️ Common Backup Mistakes

```
╔══════════════════════════════════════════════════════════════════════╗
║ MISTAKE                           │ FIX                             ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ Never testing restores            │ Monthly restore drills          ║
║ "We have backups!" (but never     │ to a staging environment        ║
║  verified they work)              │                                 ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ Backups on the SAME server        │ 3-2-1 Rule: 3 copies,          ║
║                                   │ 2 different media, 1 off-site  ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ No backup encryption              │ Encrypt all backups (AES-256)  ║
║ (stolen backup = data breach!)    │ Protect encryption keys         ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ No monitoring for backup failures │ Alert immediately on failure    ║
║ (backup job failed silently       │ Verify backup size trends       ║
║  3 weeks ago...)                  │                                 ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ Backing up only data, not config  │ Back up: data + configs +       ║
║                                   │ schemas + users/permissions     ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ No documented restore procedure   │ Write runbook with exact steps  ║
║ (DBA is on vacation when          │ Any team member should be able  ║
║  disaster strikes)                │ to restore                      ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ Ignoring backup performance       │ Use compression, parallel       ║
║ impact on production              │ backup, schedule during low     ║
║                                   │ traffic hours                   ║
╠═══════════════════════════════════╪═════════════════════════════════╣
║ No retention policy               │ Define how long to keep each    ║
║ (disk fills up, oldest backups    │ type, automate cleanup          ║
║  silently deleted)                │                                 ║
╚═══════════════════════════════════╧═════════════════════════════════╝
```

---

## 🏆 The 3-2-1 Backup Rule

```
               THE GOLDEN RULE OF BACKUPS

              ┌─────────────────────────────┐
              │         3 - 2 - 1           │
              └─────────────────────────────┘
              
     3 Copies          2 Media Types       1 Off-site
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ Original│        │  SSD    │        │  Cloud  │
   │ Database│        │         │        │ (S3/    │
   ├─────────┤        ├─────────┤        │  Azure/ │
   │ Local   │        │  NAS/   │        │  GCS)   │
   │ Backup  │        │  Tape   │        │         │
   ├─────────┤        └─────────┘        │ In a    │
   │ Off-site│                           │ DIFFERENT│
   │ Backup  │                           │ REGION   │
   └─────────┘                           └─────────┘
   
   Why 3?  One might be corrupted, one might be lost
   Why 2?  Protects against media-specific failures
   Why 1?  Flood/fire/theft at primary site? Still safe!

   Modern enhancement: 3-2-1-1-0
   + 1 immutable/air-gapped copy (ransomware protection)
   + 0 errors (verified with automated restore testing)
```

---

## 🔑 Key Takeaways

```
✅ RPO = how much data loss is acceptable; RTO = how much downtime is acceptable
✅ Full backups = simplest, largest; Incremental = smallest, complex restore chain
✅ Differential = best middle ground (Full + Differential = common strategy)
✅ Physical backups = fast, supports PITR; Logical backups = portable, selective
✅ PITR = restore to any point using base backup + transaction/WAL logs
✅ 3-2-1 Rule: 3 copies, 2 media types, 1 off-site
✅ TEST YOUR RESTORES — untested backup = no backup
✅ Monitor, alert, and document your backup procedures
✅ Encrypt backups — a stolen backup IS a data breach
✅ Choose DR tier based on business requirements (cost vs recovery speed)
```

---

## 🔗 What's Next?

**🎉 Congratulations!** You've completed **PART 1 — DATABASE FOUNDATIONS!**

You now have a rock-solid understanding of:
- What databases are and how they work internally
- Data modeling and design principles
- ACID vs BASE, CAP theorem, and distributed system trade-offs
- Indexing strategies that make queries 100x faster
- Transaction isolation and concurrency control
- Security from authentication to encryption
- Backup, recovery, and disaster planning

**Ready for PART 2?** → [SQL Databases (Relational World)](../02-SQL-Mastery/01-SQL-Basics.md)
Where we dive into SQL — the universal language of data.

---

> *"Everyone has a backup strategy. Very few have a restore strategy."* — Unknown DBA
