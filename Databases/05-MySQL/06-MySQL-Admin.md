# 2D.6 — MySQL Administration & Security 🟡

> **"A database without proper security is a ticking time bomb. A database without proper administration is a house built on sand."**
> This chapter gives you the operational skills to run MySQL like a pro — in dev, staging, AND production.

---

## 🎯 What You'll Master

```
✅ User management — creating, modifying, and dropping users
✅ Privilege system — GRANT, REVOKE, and the principle of least privilege
✅ Role-based access control (RBAC) — MySQL 8.0's game changer
✅ Authentication plugins — native, caching_sha2_password, LDAP, PAM
✅ SSL/TLS encryption — securing data in transit
✅ Data-at-rest encryption — InnoDB tablespace encryption
✅ mysqldump, mysqlpump, MySQL Shell — backup strategies
✅ Point-in-Time Recovery (PITR) — because backups alone aren't enough
✅ MySQL Shell utilities — the modern admin toolkit
✅ Monitoring, maintenance, and operational best practices
```

---

## 🔥 1. User Management — Who Gets In and What They Can Do

### Understanding MySQL's User Model

```
MySQL identifies users as: 'username'@'host'

┌──────────────────────────────────────────────────────────┐
│  'ritesh'@'localhost'     ← Only from the local machine  │
│  'ritesh'@'10.0.1.50'    ← Only from this specific IP   │
│  'ritesh'@'10.0.1.%'     ← From any IP in 10.0.1.x      │
│  'ritesh'@'%.company.com'← From any *.company.com host   │
│  'ritesh'@'%'            ← From ANYWHERE (dangerous!)   │
└──────────────────────────────────────────────────────────┘

⚠️  'ritesh'@'localhost' and 'ritesh'@'%' are DIFFERENT USERS!
    They can have different passwords and different privileges.
```

### Creating Users

```sql
-- Basic user creation
CREATE USER 'app_user'@'10.0.1.%' IDENTIFIED BY 'Str0ng_P@ssw0rd!';

-- With specific authentication plugin (MySQL 8.0 default)
CREATE USER 'app_user'@'10.0.1.%' 
    IDENTIFIED WITH caching_sha2_password BY 'Str0ng_P@ssw0rd!';

-- With password expiry
CREATE USER 'temp_user'@'%' 
    IDENTIFIED BY 'TempP@ss123'
    PASSWORD EXPIRE INTERVAL 30 DAY;

-- With connection limits
CREATE USER 'api_user'@'%' 
    IDENTIFIED BY 'AP1_P@ss!'
    WITH MAX_CONNECTIONS_PER_HOUR 1000
         MAX_USER_CONNECTIONS 50
         MAX_QUERIES_PER_HOUR 100000;

-- With account locking
CREATE USER 'dormant_user'@'%' 
    IDENTIFIED BY 'SomeP@ss'
    ACCOUNT LOCK;

-- Require SSL/TLS connection
CREATE USER 'secure_user'@'%' 
    IDENTIFIED BY 'S3cur3P@ss!'
    REQUIRE SSL;

-- Require specific SSL certificate
CREATE USER 'cert_user'@'%' 
    IDENTIFIED BY 'C3rtP@ss!'
    REQUIRE X509;
```

### Modifying Users

```sql
-- Change password
ALTER USER 'app_user'@'10.0.1.%' IDENTIFIED BY 'NewStr0ngP@ss!';

-- Force password change on next login
ALTER USER 'app_user'@'10.0.1.%' PASSWORD EXPIRE;

-- Lock/Unlock account
ALTER USER 'app_user'@'10.0.1.%' ACCOUNT LOCK;
ALTER USER 'app_user'@'10.0.1.%' ACCOUNT UNLOCK;

-- Change resource limits
ALTER USER 'app_user'@'10.0.1.%' 
    WITH MAX_CONNECTIONS_PER_HOUR 500;

-- Rename user
RENAME USER 'old_user'@'%' TO 'new_user'@'%';
```

### Dropping Users

```sql
-- Drop a user
DROP USER 'temp_user'@'%';

-- Drop if exists (no error if user doesn't exist)
DROP USER IF EXISTS 'temp_user'@'%';
```

### View All Users

```sql
-- All users
SELECT User, Host, plugin, account_locked, password_expired 
FROM mysql.user 
ORDER BY User;

-- Currently connected users
SELECT user, host, db, command, time, state 
FROM information_schema.PROCESSLIST;

-- or using Performance Schema
SELECT * FROM sys.processlist;
```

---

## 🔥 2. Privilege System — The Heart of MySQL Security

### The Privilege Hierarchy

```
┌──────────────────────────────────────────────────────────────┐
│              MYSQL PRIVILEGE LEVELS                           │
│                                                              │
│  GLOBAL (*.*)                                                │
│  ├── All databases, all tables, everything                   │
│  │   Stored in: mysql.user                                   │
│  │                                                           │
│  DATABASE (db_name.*)                                        │
│  ├── All tables in a specific database                       │
│  │   Stored in: mysql.db                                     │
│  │                                                           │
│  TABLE (db_name.table_name)                                  │
│  ├── A specific table                                        │
│  │   Stored in: mysql.tables_priv                            │
│  │                                                           │
│  COLUMN (db_name.table_name.column_name)                     │
│  ├── Specific columns in a table                             │
│  │   Stored in: mysql.columns_priv                           │
│  │                                                           │
│  ROUTINE (db_name.procedure_name)                            │
│  └── Specific stored procedures/functions                    │
│      Stored in: mysql.procs_priv                             │
└──────────────────────────────────────────────────────────────┘

💡 Privilege checking order:
   Global → Database → Table → Column → Routine
   If ANY level grants the privilege, access is allowed (OR logic)
```

### GRANT — Giving Privileges

```sql
-- ===== GLOBAL LEVEL =====
-- Full admin (like root — DANGEROUS, use sparingly)
GRANT ALL PRIVILEGES ON *.* TO 'dba_admin'@'10.0.1.%' WITH GRANT OPTION;

-- Replication user (needs only replication privileges)
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl_user'@'10.0.1.%';

-- Monitoring user
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'monitor_user'@'10.0.1.%';

-- ===== DATABASE LEVEL =====
-- Full access to a specific database
GRANT ALL PRIVILEGES ON ecommerce.* TO 'app_user'@'10.0.1.%';

-- Read-only access to a database
GRANT SELECT ON analytics.* TO 'report_user'@'10.0.1.%';

-- ===== TABLE LEVEL =====
-- Specific table privileges
GRANT SELECT, INSERT, UPDATE ON ecommerce.orders TO 'order_service'@'10.0.1.%';

-- Read-only on specific table
GRANT SELECT ON ecommerce.products TO 'catalog_reader'@'10.0.1.%';

-- ===== COLUMN LEVEL =====
-- Only see certain columns (hide sensitive data)
GRANT SELECT (id, name, email) ON ecommerce.customers TO 'support_user'@'10.0.1.%';
-- support_user CANNOT see: phone, address, credit_card columns

-- ===== ROUTINE LEVEL =====
GRANT EXECUTE ON PROCEDURE ecommerce.process_order TO 'app_user'@'10.0.1.%';
```

### REVOKE — Taking Privileges Away

```sql
-- Revoke specific privilege
REVOKE INSERT ON ecommerce.orders FROM 'order_service'@'10.0.1.%';

-- Revoke all privileges on a database
REVOKE ALL PRIVILEGES ON ecommerce.* FROM 'app_user'@'10.0.1.%';

-- Revoke all privileges everywhere
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'app_user'@'10.0.1.%';
```

### View Privileges

```sql
-- Show grants for a specific user
SHOW GRANTS FOR 'app_user'@'10.0.1.%';

-- Show grants for current user
SHOW GRANTS;
SHOW GRANTS FOR CURRENT_USER();

-- Detailed privilege check from system tables
SELECT * FROM information_schema.USER_PRIVILEGES 
WHERE GRANTEE LIKE "'app_user'%";

SELECT * FROM information_schema.SCHEMA_PRIVILEGES 
WHERE GRANTEE LIKE "'app_user'%";

SELECT * FROM information_schema.TABLE_PRIVILEGES 
WHERE GRANTEE LIKE "'app_user'%";
```

### The Principle of Least Privilege — GOLDEN RULES

```
┌──────────────────────────────────────────────────────────────┐
│        LEAST PRIVILEGE — DO THIS IN PRODUCTION               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Application user:                                        │
│     GRANT SELECT, INSERT, UPDATE, DELETE                     │
│     ON app_database.* TO 'app'@'app-server-ip';             │
│     (No CREATE, DROP, ALTER, GRANT — EVER)                   │
│                                                              │
│  ✅ Read-only reporting:                                     │
│     GRANT SELECT ON app_database.* TO 'reporter'@'...';     │
│                                                              │
│  ✅ Migration/schema user (CI/CD only):                      │
│     GRANT CREATE, ALTER, DROP, INDEX, REFERENCES             │
│     ON app_database.* TO 'migrator'@'ci-server';            │
│                                                              │
│  ✅ Backup user:                                             │
│     GRANT SELECT, RELOAD, LOCK TABLES,                       │
│     REPLICATION CLIENT, SHOW VIEW, EVENT, TRIGGER            │
│     ON *.* TO 'backup'@'backup-server';                      │
│                                                              │
│  ❌ NEVER give application users:                            │
│     ALL PRIVILEGES, SUPER, GRANT OPTION, FILE,               │
│     PROCESS, SHUTDOWN, CREATE USER                           │
│                                                              │
│  ❌ NEVER use root for application connections               │
│  ❌ NEVER use '%' for host unless truly needed               │
│  ❌ NEVER share credentials between services                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔥 3. Roles — MySQL 8.0's RBAC Revolution

Before MySQL 8.0, you had to GRANT the same set of privileges to every user individually. Roles change everything.

### Creating and Using Roles

```sql
-- Create roles (roles are like users without passwords)
CREATE ROLE 'app_read', 'app_write', 'app_admin', 'dba';

-- Grant privileges TO roles
GRANT SELECT ON ecommerce.* TO 'app_read';
GRANT INSERT, UPDATE, DELETE ON ecommerce.* TO 'app_write';
GRANT ALL PRIVILEGES ON ecommerce.* TO 'app_admin';
GRANT ALL PRIVILEGES ON *.* TO 'dba' WITH GRANT OPTION;

-- Grant roles TO users
GRANT 'app_read' TO 'report_user'@'10.0.1.%';
GRANT 'app_read', 'app_write' TO 'api_user'@'10.0.1.%';
GRANT 'app_admin' TO 'senior_dev'@'10.0.1.%';
GRANT 'dba' TO 'ritesh'@'localhost';

-- IMPORTANT: Roles must be ACTIVATED for the session
SET DEFAULT ROLE ALL TO 'api_user'@'10.0.1.%';    -- Auto-activate on login
-- or
SET DEFAULT ROLE 'app_read', 'app_write' TO 'api_user'@'10.0.1.%';

-- User can activate roles manually in their session
SET ROLE 'app_read';        -- Activate specific role
SET ROLE ALL;               -- Activate all granted roles
SET ROLE NONE;              -- Deactivate all roles
```

### Role Hierarchy — Roles within Roles

```sql
-- Create a hierarchy
CREATE ROLE 'junior_dev', 'senior_dev', 'lead_dev';

GRANT SELECT ON ecommerce.* TO 'junior_dev';
GRANT 'junior_dev' TO 'senior_dev';            -- senior inherits junior's privileges
GRANT INSERT, UPDATE, DELETE ON ecommerce.* TO 'senior_dev';
GRANT 'senior_dev' TO 'lead_dev';              -- lead inherits senior's privileges
GRANT CREATE, ALTER, DROP ON ecommerce.* TO 'lead_dev';

-- lead_dev now has: SELECT + INSERT + UPDATE + DELETE + CREATE + ALTER + DROP
```

### View Role Assignments

```sql
-- What roles does a user have?
SHOW GRANTS FOR 'api_user'@'10.0.1.%';
SHOW GRANTS FOR 'api_user'@'10.0.1.%' USING 'app_read', 'app_write';

-- What's the current user's active roles?
SELECT CURRENT_ROLE();

-- All role-user mappings
SELECT FROM_USER, FROM_HOST, TO_USER, TO_HOST 
FROM mysql.role_edges;
```

---

## 🔥 4. Authentication Plugins — How MySQL Verifies Identity

### Available Plugins

| Plugin | Description | Default In |
|--------|-------------|-----------|
| `caching_sha2_password` | SHA-256 + caching (fast & secure) | MySQL 8.0+ |
| `mysql_native_password` | SHA-1 based (legacy, widely supported) | MySQL 5.7 |
| `sha256_password` | SHA-256 (no caching, slower) | — |
| `auth_socket` / `auth_pam` | OS-level auth (Linux socket, PAM) | — |
| `authentication_ldap_simple` | LDAP authentication | Enterprise |
| `authentication_ldap_sasl` | LDAP with SASL | Enterprise |

### Changing Authentication Plugin

```sql
-- Use legacy auth for old clients that don't support SHA-256
ALTER USER 'legacy_app'@'%' 
    IDENTIFIED WITH mysql_native_password BY 'OldApp_P@ss';

-- Use modern auth (default in 8.0)
ALTER USER 'modern_app'@'%' 
    IDENTIFIED WITH caching_sha2_password BY 'M0dern_P@ss!';

-- Set default plugin for new users
-- my.cnf:
-- default_authentication_plugin = caching_sha2_password
```

> 💡 **Common Issue:** Many older MySQL clients/libraries fail to connect to MySQL 8.0 because they don't support `caching_sha2_password`. Fix: either update the client, or use `mysql_native_password` for that user (less secure but compatible).

### Password Validation Plugin — Enforce Strong Passwords

```sql
-- Install the plugin
INSTALL COMPONENT 'file://component_validate_password';

-- Configure password policy
SET GLOBAL validate_password.policy = STRONG;          -- LOW, MEDIUM, STRONG
SET GLOBAL validate_password.length = 12;              -- Minimum length
SET GLOBAL validate_password.mixed_case_count = 1;     -- At least 1 uppercase + 1 lowercase
SET GLOBAL validate_password.number_count = 1;          -- At least 1 digit
SET GLOBAL validate_password.special_char_count = 1;    -- At least 1 special char

-- Check a password strength
SELECT VALIDATE_PASSWORD_STRENGTH('MyP@ssw0rd123');
-- Returns: 100 (out of 100)

SELECT VALIDATE_PASSWORD_STRENGTH('password');
-- Returns: 25 (weak!)
```

```ini
# my.cnf — Make it permanent
[mysqld]
validate_password.policy         = STRONG
validate_password.length         = 12
validate_password.mixed_case_count = 1
validate_password.number_count   = 1
validate_password.special_char_count = 1
```

### Password History & Reuse Prevention (MySQL 8.0+)

```sql
-- Prevent reusing the last 5 passwords
ALTER USER 'app_user'@'%' PASSWORD HISTORY 5;

-- Prevent reusing passwords changed within last 365 days
ALTER USER 'app_user'@'%' PASSWORD REUSE INTERVAL 365 DAY;

-- Global setting
SET GLOBAL password_history = 5;
SET GLOBAL password_reuse_interval = 365;
```

### Failed Login Tracking & Account Locking (MySQL 8.0.19+)

```sql
-- Lock account after 3 failed login attempts for 2 days
CREATE USER 'secure_user'@'%' 
    IDENTIFIED BY 'S3cur3!'
    FAILED_LOGIN_ATTEMPTS 3
    PASSWORD_LOCK_TIME 2;        -- Days (use UNBOUNDED for permanent lock)

-- Check failed login status
SELECT User, Host, User_attributes 
FROM mysql.user 
WHERE User = 'secure_user'\G
```

---

## 🔥 5. SSL/TLS — Encrypting Data in Transit

### Why SSL/TLS Matters

```
WITHOUT SSL:
  App ──── "SELECT * FROM users WHERE id=5" ────► MySQL
       ◄── "id=5, name=Ritesh, ssn=XXX-XX-1234" ──
  
  Anyone on the network can see: queries, results, passwords 💀

WITH SSL:
  App ──── [encrypted gibberish] ────► MySQL
       ◄── [encrypted gibberish] ──
  
  Network sniffers see: nothing useful ✅
```

### Check SSL Status

```sql
-- Is SSL available?
SHOW VARIABLES LIKE '%ssl%';

-- Is current connection encrypted?
SHOW STATUS LIKE 'Ssl_cipher';
-- If empty → NOT encrypted. If shows a cipher → encrypted.

-- SSL details for current connection
\s
-- Look for "SSL: Cipher in use is TLS_AES_256_GCM_SHA384"
```

### Configure SSL (Server Side)

```ini
# my.cnf
[mysqld]
ssl_ca   = /etc/mysql/ssl/ca.pem
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key  = /etc/mysql/ssl/server-key.pem

# Require TLS 1.2+ (disable older, insecure versions)
tls_version = TLSv1.2,TLSv1.3

# Force ALL connections to use SSL
require_secure_transport = ON
```

### Generate SSL Certificates

```bash
# MySQL comes with a helper tool:
mysql_ssl_rsa_setup --datadir=/var/lib/mysql

# Or generate manually with OpenSSL:
# 1. Create CA
openssl genrsa 4096 > ca-key.pem
openssl req -new -x509 -nodes -days 3650 -key ca-key.pem -out ca.pem

# 2. Create server certificate
openssl req -newkey rsa:4096 -days 3650 -nodes -keyout server-key.pem -out server-req.pem
openssl x509 -req -in server-req.pem -days 3650 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem

# 3. Create client certificate (optional, for mutual TLS)
openssl req -newkey rsa:4096 -days 3650 -nodes -keyout client-key.pem -out client-req.pem
openssl x509 -req -in client-req.pem -days 3650 -CA ca.pem -CAkey ca-key.pem -set_serial 02 -out client-cert.pem
```

### Force SSL for Specific Users

```sql
-- Require SSL
ALTER USER 'app_user'@'%' REQUIRE SSL;

-- Require specific certificate (mutual TLS)
ALTER USER 'high_security'@'%' REQUIRE X509;

-- Require specific certificate attributes
ALTER USER 'partner_api'@'%' 
    REQUIRE SUBJECT '/CN=partner-app/O=PartnerCorp'
    AND ISSUER '/CN=MySQL-CA/O=MyCompany';
```

---

## 🔥 6. Data-at-Rest Encryption — Protecting Data on Disk

### InnoDB Tablespace Encryption (MySQL 5.7.11+)

```sql
-- Enable encryption for new tables
ALTER TABLE customers ENCRYPTION = 'Y';

-- Create encrypted table
CREATE TABLE sensitive_data (
    id INT PRIMARY KEY,
    ssn VARCHAR(11),
    credit_card VARCHAR(19)
) ENCRYPTION = 'Y';

-- Check encryption status
SELECT TABLE_SCHEMA, TABLE_NAME, CREATE_OPTIONS 
FROM information_schema.TABLES 
WHERE CREATE_OPTIONS LIKE '%ENCRYPTION%';
```

### Keyring Plugin — Managing Encryption Keys

```ini
# my.cnf
[mysqld]
# File-based keyring (simplest — for dev/small production)
early-plugin-load = keyring_file.so
keyring_file_data = /var/lib/mysql-keyring/keyring

# For enterprise/production, use:
# keyring_encrypted_file (password-protected keyring)
# keyring_vault (HashiCorp Vault integration)
# keyring_aws (AWS KMS integration)
```

### Encrypt All Tablespaces by Default

```ini
# my.cnf
[mysqld]
default_table_encryption = ON      # MySQL 8.0.16+
innodb_undo_log_encrypt  = ON      # Encrypt undo logs
innodb_redo_log_encrypt  = ON      # Encrypt redo logs
```

### Binary Log Encryption

```sql
-- Encrypt binary logs (protects replication data at rest)
SET GLOBAL binlog_encryption = ON;

-- Verify
SHOW BINARY LOGS;
-- Check the "Encrypted" column
```

---

## 🔥 7. Backup Strategies — Your Safety Net

### The Backup Landscape

```
┌──────────────────────────────────────────────────────────────┐
│                  MYSQL BACKUP METHODS                         │
│                                                              │
│  LOGICAL BACKUPS (SQL dump files)                            │
│  ├── mysqldump        — Classic, reliable, slow on large DBs │
│  ├── mysqlpump        — Parallel mysqldump (MySQL 5.7+)     │
│  └── MySQL Shell dump — Modern, fastest logical backup       │
│                                                              │
│  PHYSICAL BACKUPS (raw data files)                           │
│  ├── Percona XtraBackup — Hot backup, no locking (FREE)      │
│  ├── MySQL Enterprise Backup — Oracle's commercial solution  │
│  └── File system snapshot — LVM/ZFS/EBS snapshot             │
│                                                              │
│  POINT-IN-TIME (using binlog)                                │
│  └── Restore backup + replay binlog to exact moment          │
│                                                              │
│  ┌──────────┬────────────┬────────────┬───────────────┐      │
│  │ Method   │ Speed      │ Lock?      │ Best For      │      │
│  ├──────────┼────────────┼────────────┼───────────────┤      │
│  │mysqldump │ Slow       │ Brief lock │ < 50 GB       │      │
│  │mysqlpump │ Medium     │ Brief lock │ < 100 GB      │      │
│  │Shell dump│ Fast       │ Brief lock │ < 500 GB      │      │
│  │XtraBackup│ Fast       │ NO lock ✅ │ Any size      │      │
│  │Snapshot  │ Instant    │ Brief lock │ Any size (HA) │      │
│  └──────────┴────────────┴────────────┴───────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### mysqldump — The Classic

```bash
# Full database backup (single database)
mysqldump -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --set-gtid-purged=ON \
    ecommerce > ecommerce_backup.sql

# All databases
mysqldump -u root -p \
    --all-databases \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --set-gtid-purged=ON \
    > full_backup.sql

# Specific tables only
mysqldump -u root -p \
    --single-transaction \
    ecommerce orders products > tables_backup.sql

# Schema only (no data)
mysqldump -u root -p \
    --no-data \
    ecommerce > schema_only.sql

# Data only (no schema)
mysqldump -u root -p \
    --no-create-info \
    ecommerce > data_only.sql

# Compressed backup
mysqldump -u root -p \
    --single-transaction \
    ecommerce | gzip > ecommerce_$(date +%Y%m%d).sql.gz
```

**Critical flags explained:**

| Flag | Why It Matters |
|------|---------------|
| `--single-transaction` | Uses a consistent snapshot (InnoDB). No table locks! |
| `--routines` | Include stored procedures and functions |
| `--triggers` | Include triggers |
| `--events` | Include scheduled events |
| `--set-gtid-purged=ON` | Include GTID info (needed for GTID replication) |
| `--master-data=2` | Record binlog position (for PITR / replication setup) |
| `--flush-logs` | Rotate binlog before backup (cleaner PITR) |

### Restore from mysqldump

```bash
# Restore a single database
mysql -u root -p ecommerce < ecommerce_backup.sql

# Restore all databases
mysql -u root -p < full_backup.sql

# Restore compressed backup
gunzip < ecommerce_20240315.sql.gz | mysql -u root -p ecommerce

# Restore with progress indicator (using pv)
pv ecommerce_backup.sql | mysql -u root -p ecommerce
```

### MySQL Shell Dump & Load — The Modern Way (MySQL 8.0+)

```javascript
// MySQL Shell — MUCH faster than mysqldump (parallel, chunked)

// Dump entire instance
util.dumpInstance('/backup/full_dump', {
    threads: 8,
    compression: 'zstd',
    showProgress: true
})

// Dump specific schemas
util.dumpSchemas(['ecommerce', 'analytics'], '/backup/schema_dump', {
    threads: 8,
    compression: 'zstd'
})

// Dump specific tables
util.dumpTables('ecommerce', ['orders', 'products'], '/backup/tables_dump', {
    threads: 8
})

// Load (restore)
util.loadDump('/backup/full_dump', {
    threads: 8,
    showProgress: true,
    resetProgress: true
})
```

**MySQL Shell vs mysqldump Performance:**
```
Database Size: 50 GB

mysqldump backup:   45 minutes
Shell dump (8 threads): 8 minutes   ← 5.6x faster

mysqldump restore:  2 hours
Shell load (8 threads): 20 minutes  ← 6x faster
```

### Percona XtraBackup — Hot Physical Backup

```bash
# Full backup (NO LOCKS, NO DOWNTIME)
xtrabackup --backup \
    --target-dir=/backup/full \
    --user=backup_user \
    --password='BackupP@ss'

# Prepare the backup (apply redo logs for consistency)
xtrabackup --prepare --target-dir=/backup/full

# Restore (requires MySQL to be stopped)
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/full
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql

# Incremental backup
xtrabackup --backup \
    --target-dir=/backup/inc1 \
    --incremental-basedir=/backup/full \
    --user=backup_user \
    --password='BackupP@ss'
```

### Point-in-Time Recovery (PITR)

```
THE SCENARIO:
  Last backup: 2024-03-15 02:00 AM
  Developer drops table: 2024-03-15 10:30 AM
  You need to recover to: 2024-03-15 10:29:59 AM

STRATEGY:
  1. Restore last backup (from 02:00 AM)
  2. Replay binlogs from 02:00 AM to 10:29:59 AM
  3. Data is restored to 1 second before the disaster
```

```bash
# Step 1: Restore the base backup
mysql -u root -p < backup_20240315_0200.sql

# Step 2: Find which binlogs contain events after the backup
mysqlbinlog --start-datetime="2024-03-15 02:00:00" \
            --stop-datetime="2024-03-15 10:29:59" \
            /var/lib/mysql/binlog.000042 \
            /var/lib/mysql/binlog.000043 \
            | mysql -u root -p

# OR stop at a specific GTID (more precise)
mysqlbinlog --include-gtids="3e11fa47:100-500" \
            --exclude-gtids="3e11fa47:487" \
            /var/lib/mysql/binlog.000042 \
            | mysql -u root -p
# exclude-gtids skips the DROP TABLE transaction!
```

> 💡 **PITR is why you must ALWAYS have binlogs enabled in production.** Without binlogs, your recovery point is your last backup — could be hours or days of data loss.

---

## 🔥 8. Backup Strategy — The Production Blueprint

```
┌──────────────────────────────────────────────────────────────┐
│           PRODUCTION BACKUP STRATEGY                          │
│                                                              │
│  DAILY:                                                      │
│  └── Full backup at 2:00 AM (XtraBackup or MySQL Shell dump) │
│      • Stored locally + replicated to remote storage (S3)    │
│      • Retention: 7 days local, 30 days remote               │
│                                                              │
│  CONTINUOUS:                                                  │
│  └── Binary logs → streamed to remote storage                │
│      • Enables Point-in-Time Recovery                        │
│      • Retention: 30 days                                    │
│                                                              │
│  WEEKLY:                                                      │
│  └── Backup verification                                     │
│      • Restore to staging server                             │
│      • Run integrity checks                                  │
│      • Test PITR scenario                                    │
│                                                              │
│  REPLICA:                                                     │
│  └── Take backups from a REPLICA, not PRIMARY               │
│      • Zero impact on production performance                 │
│      • Replica is momentarily paused during backup           │
│                                                              │
│  ⚠️ GOLDEN RULE: A backup that hasn't been tested            │
│     is NOT a backup. It's just hope.                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔥 9. MySQL Shell — The Modern Admin Toolkit

MySQL Shell (`mysqlsh`) is the **next-generation** command-line client that replaces the old `mysql` CLI.

### Why MySQL Shell?

```
┌──────────────────────────────────────────────────────────────┐
│  OLD: mysql CLI                  │  NEW: MySQL Shell          │
├──────────────────────────────────┼────────────────────────────┤
│  SQL only                        │  SQL + JavaScript + Python │
│  Text output                     │  JSON, tabular, vertical   │
│  No automation                   │  Full scripting API         │
│  No cluster management           │  AdminAPI for InnoDB Cluster│
│  Basic dump/load                 │  Parallel dump/load         │
│  No upgrade checker              │  Built-in upgrade checker   │
└──────────────────────────────────┴────────────────────────────┘
```

### Essential MySQL Shell Commands

```bash
# Connect
mysqlsh root@localhost:3306

# Switch modes
\sql          # SQL mode
\js           # JavaScript mode
\py           # Python mode

# Useful commands
\status       # Connection info
\use dbname   # Switch database
\source file.sql  # Run a SQL file
\quit         # Exit
```

### Upgrade Checker (Before Major Version Upgrade)

```javascript
// Check if your server is ready for an upgrade to 8.0
util.checkForServerUpgrade('root@localhost:3306', {
    targetVersion: '8.0.36'
})

// It checks for:
// • Deprecated features you're using
// • Reserved words that became keywords
// • Authentication plugin changes
// • Removed SQL modes
// • Character set / collation issues
// • And 30+ other compatibility checks
```

---

## 🔥 10. Monitoring & Maintenance — Keeping MySQL Healthy

### Essential Monitoring Queries

```sql
-- === CONNECTION MONITORING ===
-- Current connections vs max
SELECT 
    @@max_connections AS max_allowed,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
     WHERE VARIABLE_NAME = 'Threads_connected') AS current_connections,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
     WHERE VARIABLE_NAME = 'Max_used_connections') AS peak_connections;

-- Long-running queries (potential problems)
SELECT id, user, host, db, command, time, state, 
       LEFT(info, 100) AS query_preview
FROM information_schema.PROCESSLIST 
WHERE command != 'Sleep' 
  AND time > 30
ORDER BY time DESC;

-- Kill a problematic query
KILL QUERY 12345;    -- Kill the query but keep the connection
KILL 12345;          -- Kill the entire connection

-- === TABLE MAINTENANCE ===
-- Table sizes (find your biggest tables)
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_ROWS,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS fragmented_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 20;

-- === INNODB STATUS ===
-- Quick InnoDB health check
SHOW ENGINE INNODB STATUS\G

-- === REPLICATION LAG ===
SHOW REPLICA STATUS\G
-- Check: Seconds_Behind_Source

-- === DEADLOCK DETECTION ===
-- Last deadlock info
SHOW ENGINE INNODB STATUS\G
-- Look for "LATEST DETECTED DEADLOCK" section

-- Deadlock count
SHOW STATUS LIKE 'Innodb_deadlocks';
```

### Table Maintenance Operations

```sql
-- ANALYZE TABLE — Update statistics for query optimizer
-- Run this after significant data changes
ANALYZE TABLE orders;
ANALYZE TABLE customers, products, order_items;

-- OPTIMIZE TABLE — Defragment and reclaim disk space
-- Run this after large DELETE operations
OPTIMIZE TABLE orders;
-- Note: For InnoDB, this actually does ALTER TABLE ... FORCE (rebuilds table)
-- It's an ONLINE operation in MySQL 5.6+ but takes time on large tables

-- CHECK TABLE — Verify table integrity
CHECK TABLE orders;
CHECK TABLE orders EXTENDED;    -- More thorough check

-- REPAIR TABLE — Fix corrupted MyISAM tables
-- (InnoDB self-repairs using redo/undo logs — usually no manual repair needed)
REPAIR TABLE myisam_table;
```

### Automating Maintenance with Events

```sql
-- Enable the event scheduler
SET GLOBAL event_scheduler = ON;

-- Create a weekly maintenance event
CREATE EVENT weekly_maintenance
ON SCHEDULE EVERY 1 WEEK
STARTS '2024-03-17 03:00:00'    -- Sunday 3 AM
DO
BEGIN
    -- Update table statistics
    ANALYZE TABLE ecommerce.orders;
    ANALYZE TABLE ecommerce.customers;
    ANALYZE TABLE ecommerce.products;
    
    -- Purge old data
    DELETE FROM ecommerce.audit_log 
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)
    LIMIT 100000;
END;

-- View scheduled events
SHOW EVENTS;
SELECT * FROM information_schema.EVENTS;
```

---

## 🔥 11. SQL Injection Prevention — Your #1 Security Threat

```
┌──────────────────────────────────────────────────────────────┐
│             SQL INJECTION — DEFENSE CHECKLIST                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ USE PREPARED STATEMENTS / PARAMETERIZED QUERIES          │
│     This is the #1 defense. Period.                          │
│                                                              │
│  ❌ NEVER DO THIS:                                           │
│     query = "SELECT * FROM users WHERE id = " + user_input   │
│     → Attacker sends: 1 OR 1=1; DROP TABLE users; --       │
│                                                              │
│  ✅ ALWAYS DO THIS:                                          │
│     query = "SELECT * FROM users WHERE id = ?"              │
│     stmt.execute(query, [user_input])                        │
│                                                              │
│  ADDITIONAL LAYERS:                                          │
│  ✅ Input validation (whitelist, not blacklist)               │
│  ✅ Least-privilege DB users (app can't DROP/ALTER)          │
│  ✅ Web Application Firewall (WAF)                           │
│  ✅ MySQL Enterprise Firewall (whitelist allowed queries)    │
│  ✅ Audit logging (detect suspicious queries)                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Prepared Statements in Every Language

```python
# Python (mysql-connector-python)
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# Python (SQLAlchemy)
result = session.execute(text("SELECT * FROM users WHERE id = :id"), {"id": user_id})
```

```java
// Java (JDBC)
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setInt(1, userId);
ResultSet rs = stmt.executeQuery();
```

```javascript
// Node.js (mysql2)
const [rows] = await connection.execute('SELECT * FROM users WHERE id = ?', [userId]);
```

```php
// PHP (PDO)
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([$userId]);
```

```csharp
// C# (.NET)
using var cmd = new MySqlCommand("SELECT * FROM users WHERE id = @id", conn);
cmd.Parameters.AddWithValue("@id", userId);
```

---

## 🔥 12. Audit Logging — Who Did What, When?

### MySQL Enterprise Audit (Commercial)

```ini
# my.cnf
[mysqld]
plugin-load-add = audit_log.so
audit_log_format = JSON
audit_log_policy = ALL          # LOGINS, QUERIES, ALL, NONE
audit_log_file   = /var/log/mysql/audit.log
```

### Open-Source Alternative: Percona Audit Plugin

```sql
-- Install
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- Configure
SET GLOBAL audit_log_policy = 'QUERIES';
SET GLOBAL audit_log_format = 'JSON';
```

### MariaDB Audit Plugin (Works with MySQL too)

```sql
INSTALL PLUGIN server_audit SONAME 'server_audit.so';
SET GLOBAL server_audit_logging   = ON;
SET GLOBAL server_audit_events    = 'CONNECT,QUERY,TABLE';
SET GLOBAL server_audit_file_path = '/var/log/mysql/server_audit.log';
```

---

## 🔥 13. The Security Hardening Checklist

```
┌──────────────────────────────────────────────────────────────┐
│          MYSQL SECURITY HARDENING — COMPLETE CHECKLIST        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  INSTALLATION                                                │
│  □ Run mysql_secure_installation after fresh install         │
│  □ Remove anonymous users                                    │
│  □ Remove test database                                      │
│  □ Disable remote root login                                 │
│  □ Set strong root password                                  │
│                                                              │
│  USERS & PRIVILEGES                                          │
│  □ One user per application/service                          │
│  □ Least privilege — only needed permissions                 │
│  □ No wildcards (%) for host unless required                 │
│  □ Password validation plugin enabled                        │
│  □ Password expiry policy set                                │
│  □ Failed login lockout configured                           │
│  □ Unused accounts locked or removed                         │
│                                                              │
│  NETWORK                                                     │
│  □ MySQL listens on private IP (not 0.0.0.0)                │
│  □ Firewall rules: only app servers can reach port 3306     │
│  □ SSL/TLS enabled and required                              │
│  □ require_secure_transport = ON                             │
│  □ TLS 1.2+ only (disable SSLv3, TLS 1.0, TLS 1.1)         │
│                                                              │
│  ENCRYPTION                                                  │
│  □ Data-at-rest encryption for sensitive tables              │
│  □ Binary log encryption enabled                             │
│  □ Redo/undo log encryption enabled                          │
│  □ Encryption keys managed externally (Vault/KMS)            │
│                                                              │
│  MONITORING & AUDITING                                       │
│  □ Audit logging enabled                                     │
│  □ Slow query log enabled                                    │
│  □ General log DISABLED in production (performance killer)   │
│  □ Failed login attempts monitored and alerted               │
│  □ Privilege changes tracked and reviewed                    │
│                                                              │
│  FILE SYSTEM                                                 │
│  □ MySQL data directory owned by mysql user only             │
│  □ No FILE privilege granted to app users                    │
│  □ local_infile = OFF (prevents LOAD DATA LOCAL INFILE)      │
│  □ symbolic-links = 0 (prevent symlink attacks)              │
│  □ log files readable only by root/mysql                     │
│                                                              │
│  APPLICATION LAYER                                           │
│  □ ALL queries use prepared statements                       │
│  □ Connection strings don't contain passwords in plain text  │
│  □ Use secrets management (Vault, AWS Secrets Manager, etc.) │
│  □ Connection pooling configured properly                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🧠 Quick Reference — Essential Admin Commands

```sql
-- ========== STATUS & HEALTH ==========
SHOW STATUS;                          -- Server status counters
SHOW VARIABLES;                       -- Server configuration
SHOW PROCESSLIST;                     -- Active connections
SHOW ENGINE INNODB STATUS\G           -- InnoDB detailed status
SELECT * FROM sys.processlist;        -- Better process list

-- ========== USER MANAGEMENT ==========
CREATE USER 'user'@'host' IDENTIFIED BY 'pass';
GRANT SELECT ON db.* TO 'user'@'host';
REVOKE INSERT ON db.* FROM 'user'@'host';
SHOW GRANTS FOR 'user'@'host';
DROP USER 'user'@'host';

-- ========== TABLE OPERATIONS ==========
ANALYZE TABLE table_name;             -- Update statistics
OPTIMIZE TABLE table_name;            -- Defragment & reclaim space
CHECK TABLE table_name;               -- Verify integrity

-- ========== BINARY LOGS ==========
SHOW BINARY LOGS;                     -- List all binlogs
SHOW BINARY LOG STATUS;               -- Current position
PURGE BINARY LOGS BEFORE 'date';     -- Clean old binlogs
FLUSH LOGS;                           -- Rotate current log

-- ========== REPLICATION ==========
SHOW REPLICA STATUS\G                 -- Replication health
START REPLICA;                        -- Start replication
STOP REPLICA;                         -- Stop replication

-- ========== EMERGENCY ==========
KILL QUERY thread_id;                 -- Kill a running query
KILL thread_id;                       -- Kill a connection
FLUSH PRIVILEGES;                     -- Reload privilege tables
SET GLOBAL read_only = ON;            -- Emergency read-only mode
```

---

## 🎯 Key Takeaways

```
1.  Least Privilege — Give only what's needed, nothing more
2.  Roles (MySQL 8.0) — Group privileges, assign to users, manage centrally
3.  Strong passwords — Enforce via validate_password component
4.  SSL/TLS everywhere — require_secure_transport = ON in production
5.  Encrypt at rest — InnoDB tablespace encryption for sensitive data
6.  Prepared statements — The #1 defense against SQL injection
7.  Backup + Test — Untested backups are wishful thinking
8.  PITR capability — Always have binlogs for point-in-time recovery
9.  MySQL Shell — The modern admin toolkit (dump/load, cluster mgmt, upgrade check)
10. Audit logging — Know who did what, when, from where
```

---

> **Previous:** [2D.5 — MySQL Replication & Clustering](./05-MySQL-Replication.md)
> **Next Part:** [2E — PostgreSQL](../06-PostgreSQL/01-PG-Architecture.md) — The world's most advanced open-source relational database.
