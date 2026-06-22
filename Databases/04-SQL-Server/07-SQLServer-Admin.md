# 🔧 Chapter 2C.7 — SQL Server Administration

> **Level:** 🔴 Advanced
> **Time to Master:** ~6-7 hours
> **Prerequisites:** Chapter 2C.1 (Architecture), Chapter 2C.4 (Performance Tuning), Chapter 2C.6 (HA)

> **"A good DBA is invisible — when everything runs perfectly, nobody notices. When it doesn't, everybody does."**

---

## 📌 What You'll Master

By the end of this chapter, you will:
- Manage **security** like a fortress — authentication, authorization, encryption
- Design **backup strategies** that survive any disaster
- Build **maintenance plans** that keep databases healthy for years
- Automate everything with **SQL Server Agent** (jobs, alerts, schedules)
- Monitor, troubleshoot, and **proactively prevent** problems
- Think like a **production DBA** who sleeps well at night

---

## 🎯 The DBA's World — What You're Responsible For

```
╔══════════════════════════════════════════════════════════════════╗
║                  THE DBA'S UNIVERSE                               ║
║                                                                  ║
║  ┌────────────────────────────────────────────────────────────┐  ║
║  │                                                            │  ║
║  │  🔐 SECURITY         → Who can access what?               │  ║
║  │  💾 BACKUP/RECOVERY  → Can we survive a disaster?         │  ║
║  │  🏥 MAINTENANCE      → Are databases healthy?             │  ║
║  │  ⚡ PERFORMANCE      → Is everything fast enough?         │  ║
║  │  🤖 AUTOMATION       → Are repetitive tasks automated?    │  ║
║  │  📊 MONITORING       → Do we know before users complain?  │  ║
║  │  📦 CAPACITY         → Will we run out of space?          │  ║
║  │  🔄 PATCHING         → Are we on supported versions?      │  ║
║  │  📋 COMPLIANCE       → Are we meeting audit requirements? │  ║
║  │  🚀 DEPLOYMENT       → Are changes deployed safely?       │  ║
║  │                                                            │  ║
║  └────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║  💡 A DBA who only does backups is a junior.                    ║
║     A DBA who does ALL of the above is worth their weight       ║
║     in gold (and gets paid like it).                             ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔐 PART 1 — Security (The Fortress)

### Authentication — "Who Are You?"

```
╔══════════════════════════════════════════════════════════════════╗
║               SQL SERVER AUTHENTICATION MODES                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ┌────────────────────────────────────────────────────────┐     ║
║  │  WINDOWS AUTHENTICATION (Recommended ✅)               │     ║
║  │                                                        │     ║
║  │  • Uses Active Directory credentials                   │     ║
║  │  • No password stored in SQL Server                    │     ║
║  │  • Supports Kerberos, Group Policy                     │     ║
║  │  • Centralized management                              │     ║
║  │  • Pass-through — no password prompts                 │     ║
║  │                                                        │     ║
║  │  Best for: Enterprise environments with AD             │     ║
║  └────────────────────────────────────────────────────────┘     ║
║                                                                  ║
║  ┌────────────────────────────────────────────────────────┐     ║
║  │  MIXED MODE (Windows + SQL Authentication)             │     ║
║  │                                                        │     ║
║  │  • Windows Auth (same as above) PLUS                   │     ║
║  │  • SQL Logins (username/password in SQL Server)        │     ║
║  │  • Needed for: non-Windows apps, Linux, containers     │     ║
║  │                                                        │     ║
║  │  ⚠️ SQL logins: enforce strong passwords!             │     ║
║  │  ⚠️ The 'sa' account: RENAME IT, disable it,         │     ║
║  │     or at minimum use a 30+ character password         │     ║
║  └────────────────────────────────────────────────────────┘     ║
║                                                                  ║
║  🆕 AZURE AD AUTHENTICATION (SQL 2022+)                        ║
║  └── Modern: MFA, Conditional Access, Managed Identities        ║
╚══════════════════════════════════════════════════════════════════╝
```

### Authorization — "What Can You Do?"

```
SQL SERVER SECURITY HIERARCHY:

  ┌──────────────────────────────────────────────────────┐
  │                    SQL SERVER INSTANCE                │
  │                                                      │
  │  LOGIN (Server-level principal)                      │
  │    │                                                 │
  │    │  Server Roles: sysadmin, serveradmin,           │
  │    │  securityadmin, dbcreator, public               │
  │    │                                                 │
  │    ▼                                                 │
  │  ┌──────────────────────────────────────────────┐   │
  │  │              DATABASE                         │   │
  │  │                                               │   │
  │  │  USER (Database-level principal)              │   │
  │  │    │  Mapped from a LOGIN                     │   │
  │  │    │                                          │   │
  │  │    │  Database Roles:                         │   │
  │  │    │  db_owner, db_datareader, db_datawriter, │   │
  │  │    │  db_ddladmin, db_securityadmin, public   │   │
  │  │    │                                          │   │
  │  │    ▼                                          │   │
  │  │  SCHEMA (Namespace)                           │   │
  │  │    │  dbo, Sales, HR, Finance                 │   │
  │  │    │                                          │   │
  │  │    ▼                                          │   │
  │  │  OBJECTS (Tables, Views, Procs, Functions)    │   │
  │  │    Permissions: SELECT, INSERT, UPDATE,       │   │
  │  │    DELETE, EXECUTE, ALTER, CONTROL            │   │
  │  │                                               │   │
  │  └──────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘
```

### Security Setup — Best Practices with Code

```sql
-- ═══════════════════════════════════════════════════════
-- 1. CREATE A LOGIN (Server Level)
-- ═══════════════════════════════════════════════════════

-- Windows Authentication Login
CREATE LOGIN [CORP\AppServiceAccount] FROM WINDOWS;

-- SQL Authentication Login (with strong password policy)
CREATE LOGIN AppUser 
WITH PASSWORD = 'Use_A_Str0ng_P@ssw0rd!_2026',
     DEFAULT_DATABASE = SalesDB,
     CHECK_EXPIRATION = ON,      -- Password expires per policy
     CHECK_POLICY = ON;          -- Enforce Windows password policy

-- ═══════════════════════════════════════════════════════
-- 2. CREATE A DATABASE USER (Database Level)
-- ═══════════════════════════════════════════════════════

USE SalesDB;
GO

CREATE USER [AppUser] FOR LOGIN [AppUser]
WITH DEFAULT_SCHEMA = dbo;

-- ═══════════════════════════════════════════════════════
-- 3. CREATE CUSTOM ROLES (Don't assign permissions directly!)
-- ═══════════════════════════════════════════════════════

-- Role for the application
CREATE ROLE AppRole;

-- Role for reporting
CREATE ROLE ReportRole;

-- Role for support team (limited access)
CREATE ROLE SupportRole;

-- ═══════════════════════════════════════════════════════
-- 4. GRANT PERMISSIONS TO ROLES (Principle of Least Privilege)
-- ═══════════════════════════════════════════════════════

-- App role: CRUD on specific schemas
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::Sales TO AppRole;
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::Inventory TO AppRole;
GRANT EXECUTE ON SCHEMA::Sales TO AppRole;  -- Stored procedures

-- Report role: Read-only
GRANT SELECT ON SCHEMA::Sales TO ReportRole;
GRANT SELECT ON SCHEMA::Finance TO ReportRole;

-- Support role: Very limited
GRANT SELECT ON Sales.Orders TO SupportRole;
GRANT SELECT ON Sales.Customers TO SupportRole;
-- NO access to Finance, HR, or any write operations

-- ═══════════════════════════════════════════════════════
-- 5. ADD USERS TO ROLES
-- ═══════════════════════════════════════════════════════

ALTER ROLE AppRole ADD MEMBER [AppUser];
ALTER ROLE ReportRole ADD MEMBER [CORP\ReportingTeam];
ALTER ROLE SupportRole ADD MEMBER [CORP\SupportTeam];

-- ═══════════════════════════════════════════════════════
-- 6. DENY (Overrides GRANT — blocks specific access)
-- ═══════════════════════════════════════════════════════

-- Nobody except db_owner can see salary data
DENY SELECT ON HR.EmployeeSalary TO public;

-- App cannot drop tables (even if it's in db_ddladmin somehow)
DENY ALTER ON SCHEMA::Sales TO AppRole;
```

### Row-Level Security (RLS) — Advanced

```sql
-- Scenario: Each salesperson can only see their OWN orders

-- Step 1: Create a filter function
CREATE FUNCTION Security.fn_OrderFilter(@SalesPerson VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_result
    WHERE @SalesPerson = USER_NAME()
       OR USER_NAME() = 'dbo'      -- dbo sees everything
       OR IS_MEMBER('ManagerRole') = 1;  -- Managers see everything

-- Step 2: Create a security policy
CREATE SECURITY POLICY Sales.OrderFilter
ADD FILTER PREDICATE Security.fn_OrderFilter(SalesPersonName) 
    ON Sales.Orders
WITH (STATE = ON);

-- Now when John logs in:
-- SELECT * FROM Sales.Orders → only sees John's orders!
-- When a Manager logs in → sees ALL orders!
```

### Encryption — Protecting Data at Rest & In Transit

```
╔══════════════════════════════════════════════════════════════════╗
║                SQL SERVER ENCRYPTION LAYERS                      ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  LAYER 1: TRANSPORT ENCRYPTION (Data in Transit) 🔒            ║
║  ├── TLS/SSL for connections                                    ║
║  ├── Force Encryption = ON in Configuration Manager             ║
║  └── Prevents: Network sniffing, man-in-the-middle              ║
║                                                                  ║
║  LAYER 2: TRANSPARENT DATA ENCRYPTION (TDE) — Data at Rest 🔒 ║
║  ├── Encrypts the ENTIRE database file (.mdf, .ldf)            ║
║  ├── Automatic — no app changes needed                          ║
║  ├── Prevents: Stolen disk/backup = unreadable                  ║
║  ├── Performance: ~3-5% overhead                                ║
║  └── ⚠️ Does NOT protect against: sysadmin users, SQL injection║
║                                                                  ║
║  LAYER 3: ALWAYS ENCRYPTED — Column-Level 🔒🔒                ║
║  ├── Encrypts specific columns (SSN, Credit Card)               ║
║  ├── Keys stored in client-side (Azure Key Vault, cert store)   ║
║  ├── SQL Server NEVER sees plaintext — even sysadmin can't!    ║
║  ├── Best for: PII, PHI, PCI-DSS compliance                    ║
║  └── Limitation: Can't query encrypted columns with LIKE, >    ║
║                                                                  ║
║  LAYER 4: DYNAMIC DATA MASKING — Display-Level 🎭              ║
║  ├── Masks data for non-privileged users                        ║
║  ├── Full data still stored — just hidden in query results      ║
║  ├── Masks: email(), partial(), random(), default()             ║
║  └── ⚠️ Not true encryption — clever users can bypass          ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

```sql
-- TDE Setup (Transparent Data Encryption)
-- Step 1: Create master key
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Str0ng_M@ster_Key!';

-- Step 2: Create certificate
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Certificate';

-- Step 3: Create database encryption key
USE SalesDB;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;

-- Step 4: Enable encryption
ALTER DATABASE SalesDB SET ENCRYPTION ON;

-- ⚠️ CRITICAL: Backup the certificate immediately!
BACKUP CERTIFICATE TDE_Cert
TO FILE = 'D:\Backup\TDE_Cert.cer'
WITH PRIVATE KEY (
    FILE = 'D:\Backup\TDE_Cert_Key.pvk',
    ENCRYPTION BY PASSWORD = 'Backup_P@ssword!'
);
-- If you lose this certificate, you CANNOT restore the database!

-- Dynamic Data Masking
ALTER TABLE HR.Employees
ALTER COLUMN SSN ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XX-",4)');
-- Regular user sees: XXX-XX-1234
-- Privileged user sees: 123-45-1234

ALTER TABLE HR.Employees
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
-- Regular user sees: rXXX@XXXX.com
-- Privileged user sees: ritesh@company.com

ALTER TABLE HR.Employees
ALTER COLUMN Salary ADD MASKED WITH (FUNCTION = 'random(50000, 100000)');
-- Regular user sees: random number between 50K-100K
-- Privileged user sees: actual salary

-- Grant unmask permission
GRANT UNMASK TO [CORP\HRManagers];
```

---

### SQL Server Audit — Who Did What, When?

```sql
-- Create a server audit (writes to file)
CREATE SERVER AUDIT SecurityAudit
TO FILE (
    FILEPATH = 'D:\Audits\',
    MAXSIZE = 100 MB,
    MAX_ROLLOVER_FILES = 10,
    RESERVE_DISK_SPACE = ON
)
WITH (
    QUEUE_DELAY = 1000,  -- Flush every 1 second
    ON_FAILURE = CONTINUE -- Don't stop SQL if audit fails
);

ALTER SERVER AUDIT SecurityAudit WITH (STATE = ON);

-- Audit server-level events (failed logins, permission changes)
CREATE SERVER AUDIT SPECIFICATION ServerAuditSpec
FOR SERVER AUDIT SecurityAudit
ADD (FAILED_LOGIN_GROUP),
ADD (LOGIN_CHANGE_PASSWORD_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (DATABASE_CHANGE_GROUP);

ALTER SERVER AUDIT SPECIFICATION ServerAuditSpec WITH (STATE = ON);

-- Audit database-level events (data access, schema changes)
USE SalesDB;
CREATE DATABASE AUDIT SPECIFICATION DBauditSpec
FOR SERVER AUDIT SecurityAudit
ADD (SELECT, INSERT, UPDATE, DELETE ON SCHEMA::HR BY public),
ADD (EXECUTE ON SCHEMA::HR BY public),
ADD (SCHEMA_OBJECT_CHANGE_GROUP);

ALTER DATABASE AUDIT SPECIFICATION DBauditSpec WITH (STATE = ON);

-- Read audit logs
SELECT 
    event_time,
    action_id,
    server_principal_name AS who,
    database_name,
    object_name,
    statement
FROM sys.fn_get_audit_file('D:\Audits\*.sqlaudit', DEFAULT, DEFAULT)
WHERE event_time > DATEADD(DAY, -1, GETDATE())
ORDER BY event_time DESC;
```

---

## 💾 PART 2 — Backup Strategies (Your Insurance Policy)

### Backup Types Explained

```
╔══════════════════════════════════════════════════════════════════╗
║                  SQL SERVER BACKUP TYPES                         ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  📦 FULL BACKUP — Complete snapshot of entire database          ║
║  ├── Contains: ALL data + enough log to be self-consistent      ║
║  ├── Size: Same as database size (compressed: 30-60% smaller)   ║
║  ├── Frequency: Daily or weekly                                 ║
║  └── Required: YES — the foundation of all strategies           ║
║                                                                  ║
║  📦 DIFFERENTIAL BACKUP — Changes since LAST FULL backup       ║
║  ├── Contains: Only pages changed since last FULL               ║
║  ├── Size: Grows over time until next FULL                      ║
║  ├── Frequency: Every 4-6 hours                                 ║
║  ├── Restore: FULL + latest DIFFERENTIAL                        ║
║  └── ⚠️ Cumulative, not incremental!                           ║
║                                                                  ║
║  📦 TRANSACTION LOG BACKUP — Changes since LAST LOG backup     ║
║  ├── Contains: Transaction log entries (sequential chain)       ║
║  ├── Size: Small (minutes of changes)                           ║
║  ├── Frequency: Every 5-15 minutes                              ║
║  ├── Enables: Point-in-time recovery!                           ║
║  ├── Restore: FULL + DIFF + ALL LOG backups in sequence         ║
║  └── ⚠️ ONLY works in FULL or BULK_LOGGED recovery model      ║
║      ⚠️ Break the chain = lose point-in-time recovery!        ║
║                                                                  ║
║  📦 COPY-ONLY BACKUP — Doesn't affect backup chain             ║
║  ├── Use for: Ad-hoc copies, dev refreshes, one-off needs      ║
║  └── Doesn't reset differential base or log chain              ║
║                                                                  ║
║  📦 TAIL-LOG BACKUP — Final log backup before restore          ║
║  ├── Captures: Transactions since last log backup               ║
║  ├── Critical: Take this BEFORE restoring to avoid data loss   ║
║  └── Use: BACKUP LOG ... WITH NORECOVERY                       ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### How Backup Chain Works — Visual

```
TIME →
═══════════════════════════════════════════════════════════════════

Sunday      Monday                    Tuesday
12 AM       6 AM  12 PM  6 PM        6 AM  12 PM  💥 2PM CRASH!
  │          │      │      │           │      │      │
  ▼          ▼      ▼      ▼           ▼      ▼      ▼
┌────┐   ┌────┐ ┌────┐ ┌────┐     ┌────┐ ┌────┐ ┌────┐
│FULL│   │DIFF│ │DIFF│ │DIFF│     │DIFF│ │DIFF│ │TAIL│
│    │   │  1 │ │  2 │ │  3 │     │  4 │ │  5 │ │LOG │
└────┘   └────┘ └────┘ └────┘     └────┘ └────┘ └────┘
  │                                               │
  │   + Every 15 min: LOG LOG LOG LOG LOG LOG ... LOG
  │                                               │
  └───────────────── BACKUP CHAIN ────────────────┘


TO RESTORE TO 2PM TUESDAY:
Option A (Fastest): FULL → DIFF 5 → Log backups from 12PM-2PM → Tail-log
Option B (If no diff): FULL → ALL log backups from Sunday to Tuesday 2PM
```

### Backup Commands — Complete Reference

```sql
-- ═══════════════════════════════════════════════════════
-- FULL BACKUP (with compression — always use compression!)
-- ═══════════════════════════════════════════════════════
BACKUP DATABASE SalesDB
TO DISK = 'D:\Backups\SalesDB_Full_20260602.bak'
WITH 
    COMPRESSION,                    -- 60-80% smaller
    CHECKSUM,                       -- Verify integrity
    STATS = 10,                     -- Show progress every 10%
    FORMAT,                         -- Overwrite media
    NAME = 'SalesDB Full Backup',
    DESCRIPTION = 'Weekly full backup';

-- ═══════════════════════════════════════════════════════
-- DIFFERENTIAL BACKUP
-- ═══════════════════════════════════════════════════════
BACKUP DATABASE SalesDB
TO DISK = 'D:\Backups\SalesDB_Diff_20260602_0600.bak'
WITH 
    DIFFERENTIAL,
    COMPRESSION, 
    CHECKSUM,
    STATS = 10;

-- ═══════════════════════════════════════════════════════
-- TRANSACTION LOG BACKUP (every 15 minutes)
-- ═══════════════════════════════════════════════════════
BACKUP LOG SalesDB
TO DISK = 'D:\Backups\SalesDB_Log_20260602_1415.trn'
WITH 
    COMPRESSION, 
    CHECKSUM,
    STATS = 10;

-- ═══════════════════════════════════════════════════════
-- COPY-ONLY BACKUP (doesn't break backup chain)
-- ═══════════════════════════════════════════════════════
BACKUP DATABASE SalesDB
TO DISK = 'D:\Temp\SalesDB_CopyForDev.bak'
WITH COPY_ONLY, COMPRESSION;

-- ═══════════════════════════════════════════════════════
-- BACKUP TO MULTIPLE FILES (Striping — faster!)
-- ═══════════════════════════════════════════════════════
BACKUP DATABASE SalesDB
TO DISK = 'D:\Backups\SalesDB_01.bak',
   DISK = 'E:\Backups\SalesDB_02.bak',
   DISK = 'F:\Backups\SalesDB_03.bak'
WITH COMPRESSION, CHECKSUM;
-- Writes in parallel to 3 disks — much faster for large DBs!

-- ═══════════════════════════════════════════════════════
-- BACKUP TO URL (Azure Blob Storage) — Modern approach!
-- ═══════════════════════════════════════════════════════
BACKUP DATABASE SalesDB
TO URL = 'https://mystorageaccount.blob.core.windows.net/backups/SalesDB_Full.bak'
WITH 
    CREDENTIAL = 'AzureBlobCredential',
    COMPRESSION, 
    CHECKSUM,
    MAXTRANSFERSIZE = 4194304,    -- 4MB blocks (max for Azure)
    BLOCKSIZE = 65536;
```

### Restore Scenarios — The Real Test

```sql
-- ═══════════════════════════════════════════════════════
-- SCENARIO 1: Full restore (simple)
-- ═══════════════════════════════════════════════════════
RESTORE DATABASE SalesDB
FROM DISK = 'D:\Backups\SalesDB_Full.bak'
WITH RECOVERY, REPLACE;

-- ═══════════════════════════════════════════════════════
-- SCENARIO 2: Point-in-time recovery (most common disaster recovery)
-- "Restore to Tuesday 1:55 PM — just before the bad DELETE"
-- ═══════════════════════════════════════════════════════

-- Step 1: TAIL-LOG backup (capture everything up to this moment)
BACKUP LOG SalesDB
TO DISK = 'D:\Backups\SalesDB_TailLog.trn'
WITH NORECOVERY, CHECKSUM;
-- NORECOVERY = takes DB offline for restore

-- Step 2: Restore FULL
RESTORE DATABASE SalesDB
FROM DISK = 'D:\Backups\SalesDB_Full.bak'
WITH NORECOVERY, REPLACE;

-- Step 3: Restore latest DIFFERENTIAL
RESTORE DATABASE SalesDB
FROM DISK = 'D:\Backups\SalesDB_Diff.bak'
WITH NORECOVERY;

-- Step 4: Restore LOG backups in sequence
RESTORE LOG SalesDB
FROM DISK = 'D:\Backups\SalesDB_Log_1200.trn'
WITH NORECOVERY;

RESTORE LOG SalesDB
FROM DISK = 'D:\Backups\SalesDB_Log_1215.trn'
WITH NORECOVERY;

-- Step 5: Restore last log WITH STOPAT (point-in-time!)
RESTORE LOG SalesDB
FROM DISK = 'D:\Backups\SalesDB_Log_1400.trn'
WITH STOPAT = '2026-06-02T13:55:00', RECOVERY;
-- ✅ Database is now exactly at 1:55 PM — before the disaster!

-- ═══════════════════════════════════════════════════════
-- SCENARIO 3: Restore to a DIFFERENT server (migration/clone)
-- ═══════════════════════════════════════════════════════
RESTORE DATABASE SalesDB_Copy
FROM DISK = '\\NetworkShare\SalesDB_Full.bak'
WITH 
    MOVE 'SalesDB_Data' TO 'E:\Data\SalesDB_Copy.mdf',
    MOVE 'SalesDB_Log'  TO 'F:\Logs\SalesDB_Copy_log.ldf',
    RECOVERY,
    STATS = 5;

-- ═══════════════════════════════════════════════════════
-- VERIFY backup integrity (do this regularly!)
-- ═══════════════════════════════════════════════════════
RESTORE VERIFYONLY 
FROM DISK = 'D:\Backups\SalesDB_Full.bak'
WITH CHECKSUM;
-- ✅ VERIFY ONLY — doesn't actually restore, just validates
```

### The Golden Backup Strategy

```
╔══════════════════════════════════════════════════════════════════╗
║                RECOMMENDED BACKUP STRATEGY                       ║
║                (Production — Critical Database)                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  FULL BACKUP:          Every Sunday at 12:00 AM                 ║
║  DIFFERENTIAL BACKUP:  Every day at 12:00 AM, 6:00 AM,         ║
║                        12:00 PM, 6:00 PM                        ║
║  LOG BACKUP:           Every 15 minutes (24/7)                  ║
║                                                                  ║
║  RETENTION:                                                      ║
║  ├── Local disk: 7 days                                         ║
║  ├── Network share: 30 days                                     ║
║  ├── Azure Blob Storage: 90 days                                ║
║  └── Cold storage (Archive): 7 years (compliance)               ║
║                                                                  ║
║  TESTING:                                                        ║
║  ├── Automated restore test: weekly (to a test server)          ║
║  ├── Full DR drill: quarterly                                   ║
║  └── DBCC CHECKDB: daily (after backup)                         ║
║                                                                  ║
║  RESULT:                                                         ║
║  ├── RPO: 15 minutes (worst case)                               ║
║  ├── RTO: ~30 minutes (for point-in-time recovery)              ║
║  └── Compliance: 7-year audit trail                             ║
║                                                                  ║
║  💡 THE #1 RULE: A backup you haven't tested restoring          ║
║     is NOT a backup. It's just a file.                           ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🏥 PART 3 — Maintenance (Keep It Healthy)

### The Essential Maintenance Tasks

```
╔══════════════════════════════════════════════════════════════════╗
║              DATABASE MAINTENANCE CHECKLIST                      ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DAILY:                                                          ║
║  ├── ✅ Verify backups completed successfully                   ║
║  ├── ✅ Check SQL Agent job failures                            ║
║  ├── ✅ Monitor disk space                                      ║
║  ├── ✅ Review error log for critical messages                  ║
║  └── ✅ Check AG synchronization health                        ║
║                                                                  ║
║  WEEKLY:                                                         ║
║  ├── ✅ DBCC CHECKDB (integrity check)                         ║
║  ├── ✅ Update statistics                                       ║
║  ├── ✅ Index maintenance (rebuild/reorganize)                  ║
║  ├── ✅ Cycle error logs (sp_cycle_errorlog)                    ║
║  └── ✅ Test restore from backup                                ║
║                                                                  ║
║  MONTHLY:                                                        ║
║  ├── ✅ Review index usage (find unused indexes)                ║
║  ├── ✅ Capacity planning (growth trends)                       ║
║  ├── ✅ Review security (orphaned users, excess permissions)    ║
║  └── ✅ Review wait stats and performance baselines             ║
║                                                                  ║
║  QUARTERLY:                                                      ║
║  ├── ✅ Full DR drill                                           ║
║  ├── ✅ Review patch/update status                              ║
║  └── ✅ Review and update maintenance plans                     ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### DBCC CHECKDB — Database Integrity Check

```sql
-- The most important maintenance command in SQL Server
-- Checks EVERY page in the database for corruption

-- Full check (run weekly — during low-usage window)
DBCC CHECKDB ('SalesDB') WITH NO_INFOMSGS, ALL_ERRORMSGS;

-- Physical-only check (faster — checks page-level integrity)
DBCC CHECKDB ('SalesDB') WITH PHYSICAL_ONLY, NO_INFOMSGS;

-- Check a specific table
DBCC CHECKTABLE ('Sales.Orders') WITH NO_INFOMSGS;

-- Check allocation (fast — structural check)
DBCC CHECKALLOC ('SalesDB') WITH NO_INFOMSGS;

-- ⚠️ If CHECKDB finds errors:
-- Level 1: REPAIR_REBUILD (safe — rebuilds indexes)
-- Level 2: REPAIR_ALLOW_DATA_LOSS (LAST RESORT — may lose data!)
-- 💡 ALWAYS prefer restoring from backup over REPAIR_ALLOW_DATA_LOSS
```

### Index Maintenance — Rebuild vs Reorganize

```sql
-- Check fragmentation levels
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent AS FragPercent,
    ips.page_count AS Pages,
    CASE 
        WHEN ips.avg_fragmentation_in_percent < 5  THEN 'OK — No action'
        WHEN ips.avg_fragmentation_in_percent < 30 THEN 'REORGANIZE'
        ELSE 'REBUILD'
    END AS Recommendation
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
  AND ips.page_count > 1000   -- Ignore tiny indexes
ORDER BY ips.avg_fragmentation_in_percent DESC;

/*
  FRAGMENTATION THRESHOLDS:
  ┌─────────────────────┬──────────────────────────────┐
  │  Fragmentation %    │  Action                      │
  ├─────────────────────┼──────────────────────────────┤
  │  < 5%               │  Do nothing                  │
  │  5% - 30%           │  ALTER INDEX REORGANIZE      │
  │                     │  (online, minimal locking)   │
  │  > 30%              │  ALTER INDEX REBUILD          │
  │                     │  (offline or online*)         │
  └─────────────────────┴──────────────────────────────┘
  *Online rebuild = Enterprise Edition only
*/

-- Reorganize (online — doesn't block users)
ALTER INDEX IX_Orders_Date ON Sales.Orders REORGANIZE;

-- Rebuild (offline by default — blocks users!)
ALTER INDEX IX_Orders_Date ON Sales.Orders REBUILD;

-- Rebuild ONLINE (Enterprise Edition — doesn't block users)
ALTER INDEX IX_Orders_Date ON Sales.Orders REBUILD 
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON, MAXDOP = 4);

-- Rebuild ALL indexes on a table
ALTER INDEX ALL ON Sales.Orders REBUILD 
WITH (ONLINE = ON, DATA_COMPRESSION = PAGE);

-- Update statistics after index maintenance
UPDATE STATISTICS Sales.Orders WITH FULLSCAN;
-- Or let SQL handle it:
UPDATE STATISTICS Sales.Orders;  -- Uses sampled scan
```

### Smart Index Maintenance Script

```sql
-- Automated index maintenance (the smart way)
DECLARE @TableName NVARCHAR(256), @IndexName NVARCHAR(256);
DECLARE @FragPercent FLOAT;
DECLARE @SQL NVARCHAR(MAX);

DECLARE idx_cursor CURSOR FOR
SELECT 
    QUOTENAME(OBJECT_SCHEMA_NAME(ips.object_id)) + '.' + QUOTENAME(OBJECT_NAME(ips.object_id)),
    QUOTENAME(i.name),
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
  AND ips.page_count > 1000
  AND i.name IS NOT NULL;

OPEN idx_cursor;
FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @FragPercent;

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @FragPercent < 30
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REORGANIZE;';
    ELSE
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + 
                   ' REBUILD WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);';
    
    PRINT @SQL;
    EXEC sp_executesql @SQL;
    
    FETCH NEXT FROM idx_cursor INTO @TableName, @IndexName, @FragPercent;
END;

CLOSE idx_cursor;
DEALLOCATE idx_cursor;

-- Update all statistics after maintenance
EXEC sp_updatestats;
```

> 💡 **Pro Tip (2026)**: Consider **Ola Hallengren's Maintenance Solution** ([ola.hallengren.com](https://ola.hallengren.com)) — it's the de facto industry standard for SQL Server maintenance. Free, battle-tested by thousands of enterprises.

---

## 🤖 PART 4 — SQL Server Agent (The Automator)

### What is SQL Server Agent?

```
╔══════════════════════════════════════════════════════════════════╗
║                SQL SERVER AGENT                                  ║
║                                                                  ║
║  SQL Agent = Your automated DBA assistant                        ║
║  It runs scheduled tasks while you sleep.                        ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │                                                         │    ║
║  │  JOB = A unit of work                                   │    ║
║  │    └── STEP 1: Run T-SQL (backup database)              │    ║
║  │    └── STEP 2: Run SSIS Package (ETL)                   │    ║
║  │    └── STEP 3: Run PowerShell (send Slack notification) │    ║
║  │    └── STEP 4: Run OS Command (cleanup old files)       │    ║
║  │                                                         │    ║
║  │  SCHEDULE = When to run                                 │    ║
║  │    └── Every 15 minutes / daily / weekly / monthly      │    ║
║  │                                                         │    ║
║  │  ALERT = React to events                                │    ║
║  │    └── If severity >= 17 → page the DBA                 │    ║
║  │    └── If disk < 10% → send email                       │    ║
║  │                                                         │    ║
║  │  OPERATOR = Who to notify                               │    ║
║  │    └── DBA team email, pager, net send                  │    ║
║  │                                                         │    ║
║  └─────────────────────────────────────────────────────────┘    ║
╚══════════════════════════════════════════════════════════════════╝
```

### Creating a Complete Maintenance Job

```sql
-- ═══════════════════════════════════════════════════════
-- CREATE AN OPERATOR (Who gets notified)
-- ═══════════════════════════════════════════════════════
EXEC msdb.dbo.sp_add_operator
    @name = N'DBA_Team',
    @enabled = 1,
    @email_address = N'dba-team@company.com';

-- ═══════════════════════════════════════════════════════
-- SETUP DATABASE MAIL (Required for email notifications)
-- ═══════════════════════════════════════════════════════
EXEC msdb.dbo.sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC msdb.dbo.sp_configure 'Database Mail XPs', 1;
RECONFIGURE;

-- Create mail profile and account (simplified)
EXEC msdb.dbo.sysmail_add_account_sp
    @account_name = 'SQLMailAccount',
    @email_address = 'sqlserver@company.com',
    @mailserver_name = 'smtp.company.com',
    @port = 587,
    @enable_ssl = 1;

EXEC msdb.dbo.sysmail_add_profile_sp 
    @profile_name = 'DBA_Profile';

EXEC msdb.dbo.sysmail_add_profileaccount_sp
    @profile_name = 'DBA_Profile',
    @account_name = 'SQLMailAccount',
    @sequence_number = 1;

-- ═══════════════════════════════════════════════════════
-- CREATE A NIGHTLY MAINTENANCE JOB
-- ═══════════════════════════════════════════════════════
EXEC msdb.dbo.sp_add_job
    @job_name = N'Nightly_Maintenance_SalesDB',
    @enabled = 1,
    @description = N'Nightly backup, integrity check, index maintenance',
    @owner_login_name = N'sa',
    @notify_level_email = 2,        -- Notify on failure
    @notify_email_operator_name = N'DBA_Team';

-- STEP 1: Full Backup
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Nightly_Maintenance_SalesDB',
    @step_name = N'Full Backup',
    @step_id = 1,
    @subsystem = N'TSQL',
    @command = N'
        BACKUP DATABASE SalesDB
        TO DISK = ''D:\Backups\SalesDB_Full.bak''
        WITH COMPRESSION, CHECKSUM, INIT, STATS = 10;
    ',
    @on_success_action = 3,  -- Go to next step
    @on_fail_action = 2,     -- Quit with failure
    @retry_attempts = 2,
    @retry_interval = 5;     -- Retry after 5 minutes

-- STEP 2: Integrity Check
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Nightly_Maintenance_SalesDB',
    @step_name = N'Integrity Check',
    @step_id = 2,
    @subsystem = N'TSQL',
    @command = N'DBCC CHECKDB (''SalesDB'') WITH NO_INFOMSGS, ALL_ERRORMSGS;',
    @on_success_action = 3,
    @on_fail_action = 2;

-- STEP 3: Index Maintenance
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Nightly_Maintenance_SalesDB',
    @step_name = N'Index Maintenance',
    @step_id = 3,
    @subsystem = N'TSQL',
    @command = N'
        -- Rebuild indexes with > 30% fragmentation
        EXEC sp_MSforeachtable ''
            ALTER INDEX ALL ON ? REBUILD 
            WITH (ONLINE = ON, SORT_IN_TEMPDB = ON)
        '';
        -- Update statistics
        EXEC sp_updatestats;
    ',
    @on_success_action = 3,
    @on_fail_action = 2;

-- STEP 4: Cleanup old backup files (>7 days)
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Nightly_Maintenance_SalesDB',
    @step_name = N'Cleanup Old Backups',
    @step_id = 4,
    @subsystem = N'TSQL',
    @command = N'
        EXEC master.dbo.xp_delete_files 
            ''D:\Backups\'', ''bak'', 
            DATEADD(DAY, -7, GETDATE());
    ',
    @on_success_action = 1,  -- Quit with success
    @on_fail_action = 2;

-- CREATE SCHEDULE: Every night at 1:00 AM
EXEC msdb.dbo.sp_add_jobschedule
    @job_name = N'Nightly_Maintenance_SalesDB',
    @name = N'Nightly_1AM',
    @freq_type = 4,          -- Daily
    @freq_interval = 1,      -- Every 1 day
    @active_start_time = 010000;  -- 1:00:00 AM

-- ASSIGN JOB TO LOCAL SERVER
EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'Nightly_Maintenance_SalesDB',
    @server_name = N'(local)';
```

### Creating Alerts — Proactive Monitoring

```sql
-- Alert on high-severity errors (17-25 = serious problems)
EXEC msdb.dbo.sp_add_alert
    @name = N'Critical Error Alert',
    @severity = 17,          -- Insufficient resources
    @enabled = 1,
    @delay_between_responses = 300,  -- 5 min between repeats
    @include_event_description_in = 1,
    @notification_message = N'CRITICAL: SQL Server error detected!';

EXEC msdb.dbo.sp_add_notification
    @alert_name = N'Critical Error Alert',
    @operator_name = N'DBA_Team',
    @notification_method = 1;  -- Email

-- Alert on specific errors
-- Error 823: I/O error (possible disk corruption)
EXEC msdb.dbo.sp_add_alert
    @name = N'IO Error 823',
    @message_id = 823,
    @enabled = 1;

-- Error 824: Logical I/O error (page checksum failure)
EXEC msdb.dbo.sp_add_alert
    @name = N'IO Error 824',
    @message_id = 824,
    @enabled = 1;

-- Error 825: Read retry succeeded (early warning of disk issues)
EXEC msdb.dbo.sp_add_alert
    @name = N'IO Warning 825',
    @message_id = 825,
    @enabled = 1;

-- Custom: Alert when disk space drops below 10GB
EXEC msdb.dbo.sp_add_alert
    @name = N'Low Disk Space',
    @performance_condition = N'SQLServer:Databases|Data File(s) Size (KB)|SalesDB|>|50000000',
    @enabled = 1;
```

---

## 📊 PART 5 — Monitoring (Know Before They Call You)

### Essential DMVs for Daily Monitoring

```sql
-- ═══════════════════════════════════════════════════════
-- 1. DISK SPACE CHECK
-- ═══════════════════════════════════════════════════════
SELECT 
    volume_mount_point AS Drive,
    CAST(total_bytes / 1073741824.0 AS DECIMAL(10,2)) AS TotalGB,
    CAST(available_bytes / 1073741824.0 AS DECIMAL(10,2)) AS FreeGB,
    CAST((available_bytes * 100.0 / total_bytes) AS DECIMAL(5,2)) AS FreePercent
FROM sys.dm_os_volume_stats(DB_ID(), 1);

-- ═══════════════════════════════════════════════════════
-- 2. DATABASE FILE SIZES & GROWTH
-- ═══════════════════════════════════════════════════════
SELECT 
    DB_NAME(database_id) AS DatabaseName,
    name AS FileName,
    type_desc AS FileType,
    CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS SizeMB,
    CAST(FILEPROPERTY(name, 'SpaceUsed') * 8.0 / 1024 AS DECIMAL(10,2)) AS UsedMB,
    CASE max_size 
        WHEN -1 THEN 'Unlimited'
        WHEN 0 THEN 'No Growth'
        ELSE CAST(CAST(max_size * 8.0 / 1024 AS DECIMAL(10,2)) AS VARCHAR(20)) + ' MB'
    END AS MaxSize,
    CASE is_percent_growth
        WHEN 1 THEN CAST(growth AS VARCHAR(10)) + '%'
        ELSE CAST(CAST(growth * 8.0 / 1024 AS DECIMAL(10,2)) AS VARCHAR(20)) + ' MB'
    END AS GrowthSetting
FROM sys.master_files
WHERE database_id > 4  -- Skip system databases
ORDER BY DatabaseName, type_desc;

-- ═══════════════════════════════════════════════════════
-- 3. CURRENTLY RUNNING QUERIES (What's happening RIGHT NOW?)
-- ═══════════════════════════════════════════════════════
SELECT TOP 20
    r.session_id,
    r.status,
    r.command,
    DB_NAME(r.database_id) AS DatabaseName,
    r.wait_type,
    r.wait_time / 1000.0 AS wait_seconds,
    r.cpu_time / 1000.0 AS cpu_seconds,
    r.total_elapsed_time / 1000.0 AS elapsed_seconds,
    r.reads,
    r.writes,
    r.logical_reads,
    SUBSTRING(t.text, (r.statement_start_offset/2)+1, 
        CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(t.text)
            ELSE (r.statement_end_offset - r.statement_start_offset)/2 + 1
        END) AS CurrentStatement,
    s.login_name,
    s.host_name,
    s.program_name
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50  -- Exclude system sessions
ORDER BY r.total_elapsed_time DESC;

-- ═══════════════════════════════════════════════════════
-- 4. TOP WAIT TYPES (Where is SQL Server spending time?)
-- ═══════════════════════════════════════════════════════
SELECT TOP 10
    wait_type,
    waiting_tasks_count,
    CAST(wait_time_ms / 1000.0 AS DECIMAL(12,2)) AS wait_time_seconds,
    CAST(signal_wait_time_ms / 1000.0 AS DECIMAL(12,2)) AS signal_wait_seconds,
    CAST((wait_time_ms - signal_wait_time_ms) / 1000.0 AS DECIMAL(12,2)) AS resource_wait_seconds,
    CAST(100.0 * wait_time_ms / SUM(wait_time_ms) OVER() AS DECIMAL(5,2)) AS pct
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK', 'BROKER_TO', 'SQLTRACE_BUFFER_FLUSH',
    'CLR_AUTO_EVENT', 'CLR_MANUAL_EVENT', 'LAZYWRITER_SLEEP',
    'CHECKPOINT_QUEUE', 'WAITFOR', 'XE_TIMER_EVENT',
    'BROKER_EVENTHANDLER', 'FT_IFTS_SCHEDULER_IDLE_WAIT',
    'XE_DISPATCHER_WAIT', 'SQLTRACE_INCREMENTAL_FLUSH_SLEEP',
    'HADR_FILESTREAM_IOMGR_IOCOMPLETION', 'DIRTY_PAGE_POLL',
    'SP_SERVER_DIAGNOSTICS_SLEEP'
)
AND waiting_tasks_count > 0
ORDER BY wait_time_ms DESC;

-- ═══════════════════════════════════════════════════════
-- 5. BLOCKING CHAINS (Who's blocking whom?)
-- ═══════════════════════════════════════════════════════
SELECT 
    blocked.session_id AS blocked_session,
    blocked.blocking_session_id AS blocker_session,
    DB_NAME(blocked.database_id) AS DatabaseName,
    blocked_text.text AS blocked_query,
    blocker_text.text AS blocker_query,
    blocked.wait_type,
    blocked.wait_time / 1000 AS wait_seconds,
    s_blocked.login_name AS blocked_user,
    s_blocker.login_name AS blocker_user
FROM sys.dm_exec_requests blocked
JOIN sys.dm_exec_sessions s_blocked ON blocked.session_id = s_blocked.session_id
LEFT JOIN sys.dm_exec_sessions s_blocker ON blocked.blocking_session_id = s_blocker.session_id
CROSS APPLY sys.dm_exec_sql_text(blocked.sql_handle) blocked_text
OUTER APPLY sys.dm_exec_sql_text(
    (SELECT TOP 1 most_recent_sql_handle FROM sys.dm_exec_connections 
     WHERE session_id = blocked.blocking_session_id)
) blocker_text
WHERE blocked.blocking_session_id > 0
ORDER BY blocked.wait_time DESC;

-- ═══════════════════════════════════════════════════════
-- 6. FAILED SQL AGENT JOBS (last 24 hours)
-- ═══════════════════════════════════════════════════════
SELECT 
    j.name AS JobName,
    h.step_name,
    h.run_date,
    h.run_time,
    h.run_duration,
    CASE h.run_status
        WHEN 0 THEN '❌ Failed'
        WHEN 1 THEN '✅ Succeeded'
        WHEN 2 THEN '⏸ Retry'
        WHEN 3 THEN '⚠️ Canceled'
    END AS Status,
    h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
WHERE h.run_status = 0  -- Failed
  AND h.run_date >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -1, GETDATE()), 112))
ORDER BY h.run_date DESC, h.run_time DESC;
```

---

## 🧪 Interview Questions — SQL Server Administration

### Beginner Level

```
Q1: What is the difference between Windows Authentication and SQL Authentication?
A:  Windows Auth: Uses AD credentials, no password in SQL, more secure.
    SQL Auth: Username/password stored in SQL Server, needed for 
    non-Windows clients.
    Best practice: Use Windows Auth when possible.

Q2: What are the different backup types in SQL Server?
A:  Full: Complete database snapshot (foundation)
    Differential: Changes since last FULL (cumulative)
    Transaction Log: Changes since last LOG backup (chain)
    Copy-Only: Ad-hoc backup that doesn't affect chain

Q3: What is DBCC CHECKDB and when should you run it?
A:  DBCC CHECKDB checks the physical and logical integrity of 
    every object in a database. Run it weekly (at minimum).
    It catches corruption before it becomes a disaster.
```

### Advanced Level

```
Q4: A developer accidentally ran DELETE FROM Orders (no WHERE clause)
    at 2 PM. Your last log backup was at 1:45 PM. What do you do?
A:  1. Take a tail-log backup immediately (WITH NORECOVERY)
    2. Restore FULL backup (WITH NORECOVERY)
    3. Restore latest DIFF (WITH NORECOVERY)
    4. Restore log backups in sequence (WITH NORECOVERY)
    5. Restore last log WITH STOPAT = '1:59:59 PM', RECOVERY
    → Database is now at 1 minute before the DELETE!
    Alternative: If online, use DBCC PAGE or restore to a 
    parallel DB and copy the data back.

Q5: You notice SQL Server is using 95% CPU. How do you diagnose?
A:  1. Check sys.dm_exec_requests → find high-CPU queries
    2. Check sys.dm_os_wait_stats → look for SOS_SCHEDULER_YIELD
    3. Check sys.dm_exec_query_stats → top CPU consumers
    4. Check for missing indexes (sys.dm_db_missing_index_details)
    5. Check for parameter sniffing (Query Store)
    6. Check for parallelism issues (CXPACKET waits)
    7. Check for recompilation storms

Q6: How would you implement a security audit for SOX compliance?
A:  1. Enable SQL Server Audit (server + database level)
    2. Audit all: failed logins, permission changes, DDL changes,
       access to sensitive tables (HR, Finance)
    3. Enable TDE for data-at-rest encryption
    4. Enable Always Encrypted for PII columns
    5. Implement Row-Level Security where needed
    6. Regular permission reviews (quarterly)
    7. Backup audit logs to immutable storage
    8. Set up alerts for suspicious activity
```

---

## 🗺️ DBA Daily Checklist — Quick Reference

```
╔══════════════════════════════════════════════════════════════════╗
║              DBA MORNING CHECKLIST (5 Minutes)                   ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  □ 1. Check SQL Agent jobs → any failures overnight?            ║
║  □ 2. Check backup status → all databases backed up?            ║
║  □ 3. Check disk space → any drives > 85%?                     ║
║  □ 4. Check AG health → all replicas synchronized?             ║
║  □ 5. Check error log → any severity 17+ errors?               ║
║  □ 6. Check blocking → any long-running blocked sessions?      ║
║  □ 7. Check performance → CPU/memory/IO normal?                ║
║                                                                  ║
║  If ALL green: ☕ Coffee time.                                   ║
║  If ANY red: 🔥 Fix it before users notice.                    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🎯 Chapter Summary

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ✅ SECURITY: Defense in depth                                 │
│     • Windows Auth > SQL Auth                                  │
│     • Custom roles > direct permissions                        │
│     • TDE + Always Encrypted + RLS for sensitive data          │
│     • SQL Server Audit for compliance                          │
│                                                                │
│  ✅ BACKUP: The #1 responsibility                              │
│     • FULL (weekly) + DIFF (daily) + LOG (every 15 min)        │
│     • Test restores regularly — untested = worthless           │
│     • Point-in-time recovery saves careers                     │
│                                                                │
│  ✅ MAINTENANCE: Prevention > cure                             │
│     • DBCC CHECKDB weekly (non-negotiable)                     │
│     • Index maintenance based on fragmentation levels          │
│     • Update statistics regularly                              │
│     • Ola Hallengren's solution = industry standard            │
│                                                                │
│  ✅ AUTOMATION: SQL Agent automates everything                 │
│     • Jobs, Schedules, Alerts, Operators                       │
│     • Email notifications on failure                           │
│     • Proactive alerts before problems escalate                │
│                                                                │
│  ✅ MONITORING: Know before users complain                     │
│     • DMVs are your eyes into the engine                       │
│     • Wait stats tell you WHERE time is spent                  │
│     • Morning checklist = 5 min daily routine                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2C.8 — SQL Server on Azure (Azure SQL)](./08-Azure-SQL.md) 🟡🔥
