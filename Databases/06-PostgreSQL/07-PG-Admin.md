# 2E.7 — PostgreSQL Administration 🔴

> **"A great DBA isn't the one who fixes fires — it's the one who prevents them."**
> Master PostgreSQL administration and your databases will run like a Swiss watch.

---

## 🎯 What You'll Master

```
✅ Roles, Users & Privileges — the PostgreSQL permission model
✅ Row-Level Security (RLS) — row-based access control
✅ Schemas — organizing your database objects
✅ pg_dump & pg_restore — backup and recovery like a pro
✅ Point-in-Time Recovery (PITR) — time-travel for disasters
✅ Major Version Upgrades — zero-downtime strategies
✅ Tablespaces — controlling where data lives on disk
✅ Monitoring & Alerting — the DBA's dashboard
✅ Routine Maintenance — keeping PostgreSQL healthy forever
✅ Security Hardening — locking down your database
```

---

## 👥 Roles, Users & Privileges

### Understanding the Role Model

In PostgreSQL, **there are no separate "users" and "groups"** — everything is a **role**. A role can be a user (can login), a group (contains other roles), or both.

```
┌──────────────────────────────────────────────────────────────┐
│              PostgreSQL Role Hierarchy                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SUPERUSER (postgres)                                        │
│  ├── admin_role          ← Group role (can't login)          │
│  │   ├── dba_user1       ← Login role (inherits admin privs) │
│  │   └── dba_user2                                           │
│  ├── app_role            ← Group role                        │
│  │   ├── webapp_user     ← Login role for the application    │
│  │   └── api_user        ← Login role for the API            │
│  ├── readonly_role       ← Group role                        │
│  │   ├── analyst_user    ← Login role for data analysts      │
│  │   └── grafana_user    ← Login role for monitoring         │
│  └── replication_role    ← For standby servers               │
│      └── replicator                                          │
└──────────────────────────────────────────────────────────────┘
```

### Creating Roles

```sql
-- ═══ GROUP ROLES (cannot login — used for permission bundles) ═══

-- Admin group
CREATE ROLE admin_role NOLOGIN;

-- Application group  
CREATE ROLE app_role NOLOGIN;

-- Read-only group
CREATE ROLE readonly_role NOLOGIN;


-- ═══ LOGIN ROLES (actual users) ═══

-- DBA user
CREATE ROLE dba_user1 WITH LOGIN PASSWORD 'strong_pass_here'
    CREATEDB CREATEROLE;

-- Application user (limited)
CREATE ROLE webapp_user WITH LOGIN PASSWORD 'app_pass_here'
    CONNECTION LIMIT 50;  -- Max 50 connections from this user

-- Analyst user (time-limited)
CREATE ROLE analyst_user WITH LOGIN PASSWORD 'analyst_pass'
    VALID UNTIL '2025-12-31';  -- Expires automatically

-- Replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'repl_pass';


-- ═══ GRANT GROUP MEMBERSHIP ═══

GRANT admin_role TO dba_user1;
GRANT app_role TO webapp_user, api_user;
GRANT readonly_role TO analyst_user, grafana_user;
```

### Role Attributes

| Attribute | What It Does | Default |
|-----------|-------------|---------|
| `LOGIN` / `NOLOGIN` | Can this role connect? | NOLOGIN |
| `SUPERUSER` | Bypass all permission checks? | NOSUPERUSER |
| `CREATEDB` | Can create databases? | NOCREATEDB |
| `CREATEROLE` | Can create other roles? | NOCREATEROLE |
| `REPLICATION` | Can initiate streaming replication? | NOREPLICATION |
| `INHERIT` | Automatically inherit group privileges? | INHERIT |
| `CONNECTION LIMIT n` | Max simultaneous connections | Unlimited |
| `VALID UNTIL 'date'` | Password expiration | Never |
| `BYPASSRLS` | Bypass Row-Level Security? | NOBYPASSRLS |

### Granting Privileges

```sql
-- ═══ DATABASE-LEVEL ═══
GRANT CONNECT ON DATABASE mydb TO app_role;
GRANT CREATE ON DATABASE mydb TO admin_role;

-- ═══ SCHEMA-LEVEL ═══
GRANT USAGE ON SCHEMA public TO app_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT CREATE ON SCHEMA public TO admin_role;

-- ═══ TABLE-LEVEL ═══

-- App role: full CRUD but no DDL
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_role;

-- Read-only: only SELECT
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;

-- ⚠️ The above only affects EXISTING tables. For FUTURE tables:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly_role;

-- ═══ COLUMN-LEVEL (granular!) ═══
-- Analysts can see name and email but NOT salary
GRANT SELECT (id, name, email) ON employees TO analyst_user;
-- ⚠️ They'll get "permission denied" if they SELECT salary

-- ═══ SEQUENCE-LEVEL ═══
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO app_role;

-- ═══ FUNCTION-LEVEL ═══
GRANT EXECUTE ON FUNCTION calculate_salary(int) TO app_role;
```

### Revoking Privileges

```sql
-- Remove specific privilege
REVOKE DELETE ON orders FROM app_role;

-- Remove all privileges on a table
REVOKE ALL ON orders FROM analyst_user;

-- Remove group membership
REVOKE admin_role FROM dba_user1;

-- Revoke public access (PostgreSQL grants CONNECT to PUBLIC by default!)
REVOKE CONNECT ON DATABASE mydb FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM PUBLIC;
```

> 💡 **Security Best Practice:** Always revoke from PUBLIC first, then grant specifically. PostgreSQL's default grants are too permissive for production.

### Viewing Current Privileges

```sql
-- Who has access to a table?
SELECT grantee, privilege_type
FROM information_schema.table_privileges
WHERE table_name = 'orders';

-- What roles exist and their attributes?
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin, rolreplication
FROM pg_roles
ORDER BY rolname;

-- What group memberships exist?
SELECT r.rolname AS role, m.rolname AS member
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member;

-- Short form (psql):
-- \du          → List all roles
-- \dp orders   → Show privileges on 'orders' table
-- \dn+         → Show schemas with privileges
```

---

## 🔒 Row-Level Security (RLS) — Row-Based Access Control

RLS lets you control **which rows** a user can see or modify. Not just which tables — which rows within a table.

### The Problem RLS Solves

```
Without RLS:
  SELECT * FROM orders;
  → Every user sees ALL orders from ALL customers.
  → The application must filter: WHERE tenant_id = current_user_tenant()
  → If a developer forgets the filter → DATA LEAK!

With RLS:
  SELECT * FROM orders;
  → PostgreSQL automatically filters rows based on the user's identity.
  → Even if the developer forgets to filter → the database enforces it.
  → Defense in depth. Security at the database level.
```

### Setting Up RLS

```sql
-- Step 1: Enable RLS on the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Step 2: Create policies (rules for who sees what)

-- Tenants can only see their own orders
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::int);

-- The USING clause is evaluated for every row.
-- If it returns TRUE → row is visible. FALSE → row is invisible.

-- Step 3: Set the tenant context in your application
SET app.tenant_id = '42';  -- This user belongs to tenant 42
SELECT * FROM orders;       -- Only sees orders where tenant_id = 42
```

### RLS Policy Types

```sql
-- ═══ SELECT Policy (controls reads) ═══
CREATE POLICY read_own_data ON orders
    FOR SELECT
    USING (user_id = current_user_id());

-- ═══ INSERT Policy (controls what you can insert) ═══
CREATE POLICY insert_own_data ON orders
    FOR INSERT
    WITH CHECK (user_id = current_user_id());
    -- WITH CHECK validates new rows being inserted

-- ═══ UPDATE Policy (controls which rows can be updated AND what the new values can be) ═══
CREATE POLICY update_own_data ON orders
    FOR UPDATE
    USING (user_id = current_user_id())          -- Which rows can be selected for update
    WITH CHECK (user_id = current_user_id());     -- What the updated row must satisfy

-- ═══ DELETE Policy ═══
CREATE POLICY delete_own_data ON orders
    FOR DELETE
    USING (user_id = current_user_id());

-- ═══ ALL Operations ═══
CREATE POLICY full_access_own ON orders
    FOR ALL
    USING (user_id = current_user_id())
    WITH CHECK (user_id = current_user_id());
```

### Multi-Tenant SaaS Example

```sql
-- The classic multi-tenant pattern

-- 1. Table with tenant_id column
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,
    title TEXT,
    content TEXT,
    created_by TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- 2. Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- 3. Tenant isolation policy
CREATE POLICY tenant_isolation ON documents
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::int)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::int);

-- 4. Admin bypass — admins see everything
CREATE POLICY admin_bypass ON documents
    FOR ALL
    TO admin_role
    USING (true)        -- See all rows
    WITH CHECK (true);  -- Modify any row

-- 5. Application usage:
-- At the start of each request, set the tenant context:
SET app.tenant_id = '42';

-- Now ALL queries on 'documents' are automatically filtered!
SELECT * FROM documents;
-- SQL executes as: SELECT * FROM documents WHERE tenant_id = 42;
-- Even JOINs, CTEs, subqueries — ALL filtered. Zero leaks possible.
```

### RLS Best Practices

```
✅ DO:
  • Use RLS for multi-tenant applications (defense in depth)
  • Combine with application-level filtering (belt AND suspenders)
  • Test with different roles — don't just test as superuser
  • Use FORCE ROW LEVEL SECURITY for table owners too:
      ALTER TABLE orders FORCE ROW LEVEL SECURITY;
  • Create separate policies for different roles (don't over-complicate one policy)

❌ DON'T:
  • Rely solely on RLS (still validate in application code)
  • Forget that superusers and table owners bypass RLS by default
  • Use expensive functions in USING clauses (evaluated per row!)
  • Forget to create policies after enabling RLS
    (if no policy exists → ALL rows are denied! ← Common mistake)
```

---

## 📁 Schemas — Organizing Your Database

Schemas are **namespaces** within a database. Think of them like folders for your tables, views, functions, etc.

```sql
-- Create schemas
CREATE SCHEMA sales;
CREATE SCHEMA hr;
CREATE SCHEMA analytics;

-- Create tables in specific schemas
CREATE TABLE sales.orders (
    id SERIAL PRIMARY KEY,
    amount NUMERIC(12,2)
);

CREATE TABLE hr.employees (
    id SERIAL PRIMARY KEY,
    name TEXT,
    salary NUMERIC(10,2)
);

-- Query with schema prefix
SELECT * FROM sales.orders;
SELECT * FROM hr.employees;

-- Set default search path (so you don't have to prefix)
ALTER ROLE webapp_user SET search_path = sales, public;

-- Grant schema access
GRANT USAGE ON SCHEMA sales TO app_role;
GRANT USAGE ON SCHEMA analytics TO readonly_role;
```

### Schema Use Cases

```
1. MULTI-TENANT ISOLATION (schema-per-tenant)
   tenant_1.orders, tenant_2.orders, tenant_3.orders
   Each tenant has identical table structures in separate schemas.

2. LOGICAL SEPARATION
   sales.orders, hr.employees, inventory.products
   Different departments, different schemas.

3. VERSION MANAGEMENT
   v1.api_users, v2.api_users
   Different API versions with different table structures.

4. EXTENSION ISOLATION
   extensions.postgis_data, extensions.timescale_data
   Keep extension tables out of the main schema.
```

---

## 💾 Backup & Recovery — pg_dump / pg_restore

### Backup Strategies Overview

```
┌──────────────────────────────────────────────────────────────┐
│              PostgreSQL Backup Methods                        │
├──────────────┬──────────────────────────────────────────────┤
│  pg_dump     │ Logical backup. Exports SQL or custom format.│
│              │ Per-database. Good for small/medium DBs.     │
├──────────────┼──────────────────────────────────────────────┤
│  pg_dumpall  │ Logical backup of ALL databases + globals    │
│              │ (roles, tablespaces). Full cluster backup.   │
├──────────────┼──────────────────────────────────────────────┤
│  pg_basebackup│ Physical backup. Copies entire data dir.   │
│              │ Required for PITR. Good for large DBs.      │
├──────────────┼──────────────────────────────────────────────┤
│  WAL Archiving│ Continuous archiving of WAL files.          │
│ + pg_basebackup│ Combined = Point-in-Time Recovery (PITR). │
└──────────────┴──────────────────────────────────────────────┘
```

### pg_dump — Logical Backup

```bash
# ═══ FORMATS ═══

# 1. Plain SQL (human-readable, slow restore)
pg_dump -h localhost -U postgres mydb > backup.sql

# 2. Custom format (compressed, parallel restore, RECOMMENDED)
pg_dump -h localhost -U postgres -Fc mydb > backup.dump

# 3. Directory format (parallel dump AND restore)
pg_dump -h localhost -U postgres -Fd -j 4 mydb -f backup_dir/
#   -j 4 = use 4 parallel workers

# 4. Tar format
pg_dump -h localhost -U postgres -Ft mydb > backup.tar
```

### pg_dump Options Cheat Sheet

```bash
# ═══ COMMON OPTIONS ═══

# Backup specific tables
pg_dump -Fc -t orders -t customers mydb > tables.dump

# Backup a specific schema
pg_dump -Fc -n sales mydb > sales_schema.dump

# Exclude a table
pg_dump -Fc -T audit_logs mydb > no_audit.dump

# Schema only (no data)
pg_dump -Fc --schema-only mydb > schema.dump

# Data only (no DDL)
pg_dump -Fc --data-only mydb > data.dump

# Include CREATE DATABASE statement
pg_dump -Fc --create mydb > backup_with_create.dump

# Compress (custom format is already compressed, but for plain SQL):
pg_dump mydb | gzip > backup.sql.gz
```

### pg_restore — Restore from Backup

```bash
# ═══ RESTORE FROM CUSTOM FORMAT ═══

# Restore into existing database
pg_restore -h localhost -U postgres -d mydb backup.dump

# Restore with parallel workers (FAST!)
pg_restore -h localhost -U postgres -d mydb -j 4 backup.dump

# Restore specific tables only
pg_restore -h localhost -U postgres -d mydb -t orders -t customers backup.dump

# Restore schema only
pg_restore -h localhost -U postgres -d mydb --schema-only backup.dump

# List contents of a backup (see what's inside)
pg_restore --list backup.dump

# Clean (drop objects before recreating)
pg_restore -h localhost -U postgres -d mydb --clean --if-exists backup.dump

# ═══ RESTORE FROM PLAIN SQL ═══
psql -h localhost -U postgres -d mydb < backup.sql
# Or compressed:
gunzip -c backup.sql.gz | psql -h localhost -U postgres -d mydb
```

### pg_dumpall — Backup Everything

```bash
# Backup ALL databases + global objects (roles, tablespaces)
pg_dumpall -h localhost -U postgres > full_cluster.sql

# Backup only global objects (roles, tablespaces) — pair with per-DB pg_dumps
pg_dumpall -h localhost -U postgres --globals-only > globals.sql
```

### Backup Strategy for Production

```bash
#!/bin/bash
# production_backup.sh — Run daily via cron

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/postgresql"
RETENTION_DAYS=30

# 1. Globals (roles, tablespaces) — small, always dump
pg_dumpall -U postgres --globals-only > ${BACKUP_DIR}/globals_${DATE}.sql

# 2. Each database in custom format (parallel)
for DB in $(psql -U postgres -At -c "SELECT datname FROM pg_database WHERE datistemplate = false"); do
    pg_dump -U postgres -Fd -j 4 ${DB} -f ${BACKUP_DIR}/${DB}_${DATE}/
done

# 3. Cleanup old backups
find ${BACKUP_DIR} -type f -mtime +${RETENTION_DAYS} -delete
find ${BACKUP_DIR} -type d -empty -delete

# 4. Verify backup integrity
pg_restore --list ${BACKUP_DIR}/mydb_${DATE}/ > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "ALERT: Backup verification failed!" | mail -s "PG Backup Failed" dba@company.com
fi
```

---

## ⏰ Point-in-Time Recovery (PITR) — Time Travel

PITR lets you restore your database to **any specific point in time** — e.g., "Restore to 2 minutes before that accidental DROP TABLE at 3:47 PM."

### How PITR Works

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Base Backup        WAL Archives (continuous)                │
│  (full snapshot)    (every change since backup)              │
│                                                              │
│  ████████████  →  [WAL1] [WAL2] [WAL3] [WAL4] [WAL5]       │
│  Jan 1, 2am       Jan 1   Jan 1  Jan 2  Jan 2  Jan 2       │
│                   3am     4am    1am    2am    3am          │
│                                                              │
│  Restore target: Jan 2, 1:30 AM                              │
│  1. Restore base backup (Jan 1, 2am)                         │
│  2. Replay WAL1, WAL2, WAL3                                  │
│  3. Replay WAL4 up to 1:30 AM only                           │
│  4. Stop. Database is now at Jan 2, 1:30 AM ✅               │
│                                                              │
│  Everything after 1:30 AM (the bad DROP TABLE) never happened│
└──────────────────────────────────────────────────────────────┘
```

### Setting Up WAL Archiving

```ini
# postgresql.conf

# Enable archiving
archive_mode = on
archive_command = 'cp %p /archive/wal/%f'
# %p = path to WAL file
# %f = filename of WAL file

# For cloud storage:
# archive_command = 'aws s3 cp %p s3://my-bucket/wal/%f'

# For production, use pgBackRest or WAL-G instead of raw cp:
# archive_command = 'pgbackrest --stanza=mydb archive-push %p'
```

### Taking a Base Backup for PITR

```bash
# Physical base backup (the starting point for PITR)
pg_basebackup -h localhost -U postgres \
    -D /backups/base_$(date +%Y%m%d) \
    -Ft -z -Xs -P

# -Ft = tar format
# -z  = compress with gzip
# -Xs = stream WAL during backup
# -P  = show progress
```

### Performing PITR

```bash
# 1. Stop PostgreSQL
sudo systemctl stop postgresql

# 2. Move current data directory aside
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main_bad

# 3. Restore the base backup
tar xzf /backups/base_20240115/base.tar.gz -C /var/lib/postgresql/16/main/

# 4. Create recovery configuration
cat > /var/lib/postgresql/16/main/postgresql.auto.conf << EOF
restore_command = 'cp /archive/wal/%f %p'
recovery_target_time = '2024-01-15 15:45:00'
recovery_target_action = 'promote'
EOF

# 5. Create the recovery signal file
touch /var/lib/postgresql/16/main/recovery.signal

# 6. Start PostgreSQL — it will replay WAL up to the target time
sudo systemctl start postgresql

# 7. Verify
psql -c "SELECT pg_is_in_recovery();"  -- Should be false (promoted)
psql -c "SELECT * FROM accidentally_dropped_table;"  -- It's back! 🎉
```

### Recovery Target Options

```ini
# Recover to a specific time
recovery_target_time = '2024-01-15 15:45:00+05:30'

# Recover to a specific transaction ID
recovery_target_xid = '12345678'

# Recover to a named restore point
recovery_target_name = 'before_migration'
-- Created earlier with: SELECT pg_create_restore_point('before_migration');

# Recover to the end of a specific WAL file
recovery_target_lsn = '0/1A2B3C4D'

# What to do after reaching the target:
recovery_target_action = 'promote'   -- Become a primary (most common)
recovery_target_action = 'pause'     -- Pause and let you inspect before committing
recovery_target_action = 'shutdown'  -- Shut down cleanly
```

> 💡 **Pro Tip:** Before risky operations (migrations, bulk deletes), create a named restore point:
> ```sql
> SELECT pg_create_restore_point('before_scary_migration_v2');
> ```
> This gives you a named target to recover to if things go wrong.

---

## 🔄 Major Version Upgrades

PostgreSQL releases a new major version every year. Upgrading is essential for security patches, performance improvements, and new features.

### Upgrade Methods

```
┌─────────────────────────────────────────────────────────────────┐
│              PostgreSQL Upgrade Strategies                       │
├──────────────┬──────────────────────────────────────────────────┤
│  pg_upgrade  │ In-place upgrade. Fast. Minutes of downtime.    │
│   (pg_upgrade │ Links/copies data files. Requires same OS.     │
│    --link)   │ Recommended for most upgrades.                  │
├──────────────┼──────────────────────────────────────────────────┤
│  pg_dump /   │ Export and reimport. Works across any versions. │
│  pg_restore  │ Hours of downtime for large databases.          │
├──────────────┼──────────────────────────────────────────────────┤
│  Logical     │ Set up PG 14 → PG 16 logical replication.      │
│  Replication │ Switch traffic when in sync. Near-zero downtime.│
│              │ Best for large databases that can't afford hours.│
├──────────────┼──────────────────────────────────────────────────┤
│  Patroni +   │ Rolling upgrade across cluster nodes.           │
│  pg_upgrade  │ Zero downtime with proper orchestration.        │
└──────────────┴──────────────────────────────────────────────────┘
```

### pg_upgrade — The Standard Approach

```bash
# ═══ PRE-UPGRADE CHECKLIST ═══

# 1. Check for incompatibilities
/usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/var/lib/postgresql/14/main \
    --new-datadir=/var/lib/postgresql/16/main \
    --old-bindir=/usr/lib/postgresql/14/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --check  # ← DRY RUN — checks without doing anything

# 2. Review output for warnings about:
#    - Extensions that need updating
#    - Data types that changed
#    - Configuration changes needed


# ═══ PERFORM THE UPGRADE ═══

# Stop both PostgreSQL versions
sudo systemctl stop postgresql@14-main
sudo systemctl stop postgresql@16-main

# Run pg_upgrade with --link (fast: creates hard links instead of copying)
/usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/var/lib/postgresql/14/main \
    --new-datadir=/var/lib/postgresql/16/main \
    --old-bindir=/usr/lib/postgresql/14/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --link   # ← Use hard links (seconds instead of hours)

# Start the new version
sudo systemctl start postgresql@16-main

# ═══ POST-UPGRADE ═══

# 1. Update statistics (CRITICAL — planner needs fresh stats)
/usr/lib/postgresql/16/bin/vacuumdb --all --analyze-in-stages

# 2. Run the generated delete script (removes old cluster data)
# ./delete_old_cluster.sh    ← Only after verifying everything works!

# 3. Update extensions
psql -c "ALTER EXTENSION pg_stat_statements UPDATE;"
psql -c "ALTER EXTENSION postgis UPDATE;"
```

### Logical Replication Upgrade (Zero Downtime)

```
Step-by-step for upgrading PG 14 → PG 16 with zero downtime:

1. Install PG 16 on a new server (or same server, different port)
2. Create the same schema on PG 16 (pg_dump --schema-only)
3. Create publication on PG 14:
   CREATE PUBLICATION upgrade_pub FOR ALL TABLES;
4. Create subscription on PG 16:
   CREATE SUBSCRIPTION upgrade_sub CONNECTION '...' PUBLICATION upgrade_pub;
5. Wait for initial sync to complete
6. Monitor replication lag until it's near zero
7. Stop writes to PG 14 (set read-only or stop application)
8. Wait for lag = 0 (all data caught up)
9. Switch application connection strings to PG 16
10. Drop subscription and publication
11. Done! PG 16 is live. Total write downtime: seconds.
```

---

## 🔐 Security Hardening

### pg_hba.conf — Client Authentication

This file controls **who can connect, from where, and how they authenticate**.

```ini
# pg_hba.conf — Read top to bottom, first matching rule wins

# TYPE   DATABASE   USER         ADDRESS           METHOD

# Local connections (Unix socket)
local   all        postgres                       peer       # OS user = PG user
local   all        all                            scram-sha-256

# Localhost (TCP)
host    all        all          127.0.0.1/32      scram-sha-256
host    all        all          ::1/128           scram-sha-256

# Application servers (specific subnet)
host    mydb       webapp_user  10.0.1.0/24       scram-sha-256
host    mydb       api_user     10.0.2.0/24       scram-sha-256

# Replication (standby servers)
host    replication replicator  10.0.3.0/24       scram-sha-256

# Analysts (specific IPs only)
host    mydb       analyst_user 10.10.5.100/32    scram-sha-256

# DENY everything else (implicit — no matching rule = reject)
# But you can be explicit:
host    all        all          0.0.0.0/0         reject
```

### Authentication Methods

| Method | Security | Use When |
|--------|----------|----------|
| `scram-sha-256` | 🟢 Strongest | ✅ Default for all remote connections |
| `md5` | 🟡 Legacy | Older clients that don't support SCRAM |
| `peer` | 🟢 Strong (local only) | Local connections — OS user = PG user |
| `cert` | 🟢 Strongest | Mutual TLS authentication |
| `ldap` | 🟢 Strong | Active Directory / LDAP integration |
| `gss` | 🟢 Strong | Kerberos (Windows AD) |
| `trust` | 🔴 NONE | ⚠️ NEVER in production (no password required!) |
| `reject` | N/A | Explicitly deny connections |

### SSL/TLS Configuration

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'        # For client cert verification
ssl_min_protocol_version = 'TLSv1.2'          # Reject TLS 1.0/1.1

# Force SSL for all remote connections (pg_hba.conf):
hostssl  all  all  0.0.0.0/0  scram-sha-256    # hostSSL = require SSL
```

### Security Checklist

```
□  1. Change default postgres password
□  2. Revoke unnecessary PUBLIC privileges:
       REVOKE ALL ON SCHEMA public FROM PUBLIC;
       REVOKE CONNECT ON DATABASE mydb FROM PUBLIC;
□  3. Use scram-sha-256 authentication (not md5 or trust)
□  4. Enable SSL/TLS for all remote connections
□  5. Restrict pg_hba.conf to specific IP ranges
□  6. Use separate roles for application, analytics, admin
□  7. Enable Row-Level Security for multi-tenant apps
□  8. Enable pg_audit extension for audit logging
□  9. Set password expiration: VALID UNTIL for temporary users
□ 10. Use connection limits: CONNECTION LIMIT per role
□ 11. Disable superuser login from remote (only allow local peer)
□ 12. Keep PostgreSQL updated (security patches)
□ 13. Set log_connections = on, log_disconnections = on
□ 14. Set log_statement = 'ddl' (log all DDL changes)
□ 15. Encrypt backups at rest
```

---

## 📊 Monitoring & Observability

### The DBA's Dashboard — Essential Queries

```sql
-- ═══ 1. DATABASE SIZE ═══
SELECT 
    datname AS database,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;

-- ═══ 2. TABLE SIZES (top 20) ═══
SELECT 
    schemaname || '.' || tablename AS table,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS data_size,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename) - 
        pg_relation_size(schemaname || '.' || tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- ═══ 3. ACTIVE CONNECTIONS ═══
SELECT 
    datname,
    usename,
    state,
    COUNT(*) AS count
FROM pg_stat_activity
GROUP BY datname, usename, state
ORDER BY count DESC;

-- ═══ 4. CONNECTION USAGE ═══
SELECT 
    max_conn,
    used,
    max_conn - used AS available,
    round(100.0 * used / max_conn, 1) AS pct_used
FROM 
    (SELECT count(*) AS used FROM pg_stat_activity) t,
    (SELECT setting::int AS max_conn FROM pg_settings WHERE name = 'max_connections') s;

-- ═══ 5. SLOW QUERIES (currently running) ═══
SELECT 
    pid,
    now() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%'
  AND now() - query_start > interval '5 seconds'
ORDER BY duration DESC;

-- ═══ 6. BLOAT (dead tuples) ═══
SELECT 
    schemaname || '.' || relname AS table,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS bloat_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;

-- ═══ 7. INDEX HIT RATE (should be > 95%) ═══
SELECT 
    relname AS table,
    CASE WHEN seq_scan + idx_scan = 0 THEN 0
         ELSE round(100.0 * idx_scan / (seq_scan + idx_scan), 1)
    END AS idx_hit_pct,
    seq_scan,
    idx_scan
FROM pg_stat_user_tables
WHERE (seq_scan + idx_scan) > 100
ORDER BY idx_hit_pct ASC
LIMIT 20;

-- ═══ 8. CACHE HIT RATE (should be > 99%) ═══
SELECT 
    sum(heap_blks_hit) AS cache_hits,
    sum(heap_blks_read) AS disk_reads,
    round(100.0 * sum(heap_blks_hit) / 
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS cache_hit_pct
FROM pg_statio_user_tables;

-- ═══ 9. REPLICATION LAG ═══
SELECT 
    client_addr,
    application_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag
FROM pg_stat_replication;

-- ═══ 10. LOCKS AND BLOCKING ═══
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    LEFT(blocked_activity.query, 60) AS blocked_query,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    LEFT(blocking_activity.query, 60) AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### Setting Up Logging

```ini
# postgresql.conf — Logging Configuration

# Where to log
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d              # New log file daily
log_rotation_size = 100MB          # Or when file reaches 100MB

# What to log
log_min_duration_statement = 500   # Log queries slower than 500ms
log_statement = 'ddl'              # Log all DDL (CREATE, ALTER, DROP)
log_connections = on               # Log every connection
log_disconnections = on            # Log every disconnection
log_lock_waits = on                # Log when waiting for locks > deadlock_timeout
log_checkpoints = on               # Log checkpoint activity
log_temp_files = 0                 # Log all temp file usage
log_autovacuum_min_duration = 250  # Log autovacuum runs > 250ms

# Format
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
# %t = timestamp, %p = PID, %u = user, %d = database, %a = application, %h = client host
```

### Prometheus + Grafana Stack

```yaml
# docker-compose.yml for PostgreSQL monitoring

services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:password@postgres:5432/mydb?sslmode=disable"
    ports:
      - "9187:9187"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    # Import dashboard: grafana.com/grafana/dashboards/9628
```

---

## 🔧 Routine Maintenance

### Daily Tasks (Automated)

```sql
-- These should be handled by autovacuum, but verify:

-- Check autovacuum is running
SELECT name, setting FROM pg_settings WHERE name LIKE 'autovacuum%';

-- Check tables that haven't been vacuumed recently
SELECT 
    schemaname || '.' || relname AS table,
    last_autovacuum,
    last_autoanalyze,
    n_dead_tup
FROM pg_stat_user_tables
WHERE last_autovacuum IS NULL 
   OR last_autovacuum < now() - interval '7 days'
ORDER BY n_dead_tup DESC;
```

### Weekly Tasks

```sql
-- 1. Check for unused indexes (wasting disk + slowing writes)
SELECT 
    schemaname || '.' || indexrelname AS index,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan AS scans_since_last_stats_reset
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint)
ORDER BY pg_relation_size(indexrelid) DESC;

-- 2. Check for duplicate indexes
SELECT 
    a.indexrelid::regclass AS index1,
    b.indexrelid::regclass AS index2,
    a.indrelid::regclass AS table
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid 
    AND a.indexrelid < b.indexrelid
    AND a.indkey = b.indkey;

-- 3. Check for tables approaching transaction ID wraparound
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    current_setting('autovacuum_freeze_max_age')::bigint AS freeze_max,
    round(100.0 * age(datfrozenxid) / 
        current_setting('autovacuum_freeze_max_age')::bigint, 1) AS pct_towards_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
-- ⚠️ If pct_towards_wraparound > 75%, investigate immediately!
```

### Monthly Tasks

```bash
# 1. Verify backups by restoring to a test server
pg_restore -h test-server -U postgres -d test_restore -j 4 /backups/latest.dump
# Run some validation queries on test_restore

# 2. Review and rotate logs
find /var/log/postgresql -name "*.log" -mtime +30 -delete

# 3. Check disk space trends
df -h /var/lib/postgresql

# 4. Review pg_stat_statements for new slow queries
# (see Chapter 2E.5 for queries)

# 5. Check for extension updates
psql -c "SELECT name, default_version, installed_version 
         FROM pg_available_extensions 
         WHERE installed_version IS NOT NULL 
           AND default_version != installed_version;"
```

---

## 🗄️ Tablespaces — Controlling Data Location

Tablespaces let you place tables and indexes on **different physical disks**.

```sql
-- Create a tablespace on a fast SSD
CREATE TABLESPACE fast_ssd LOCATION '/mnt/ssd/pg_data';

-- Create a tablespace on a large HDD (for archive data)
CREATE TABLESPACE bulk_hdd LOCATION '/mnt/hdd/pg_archive';

-- Put hot tables on SSD
ALTER TABLE orders SET TABLESPACE fast_ssd;
ALTER INDEX idx_orders_date SET TABLESPACE fast_ssd;

-- Put archive/cold tables on HDD
ALTER TABLE orders_archive SET TABLESPACE bulk_hdd;

-- Set default tablespace for new objects
SET default_tablespace = 'fast_ssd';

-- Check tablespace usage
SELECT 
    spcname AS tablespace,
    pg_size_pretty(pg_tablespace_size(spcname)) AS size
FROM pg_tablespace;
```

---

## 🧰 Essential psql Commands

```
 Command              │ What It Does
──────────────────────┼────────────────────────────────────
 \l                   │ List all databases
 \c dbname            │ Connect to a database
 \dt                  │ List tables in current schema
 \dt+                 │ List tables with sizes
 \di                  │ List indexes
 \dv                  │ List views
 \df                  │ List functions
 \du                  │ List roles/users
 \dn                  │ List schemas
 \dp tablename        │ Show table privileges
 \d tablename         │ Describe table structure
 \d+ tablename        │ Describe table with extra info
 \x                   │ Toggle expanded display
 \timing              │ Toggle query timing
 \e                   │ Open query in $EDITOR
 \i filename.sql      │ Execute SQL file
 \copy                │ Client-side COPY (no server perms needed)
 \watch 5             │ Re-run last query every 5 seconds
 \conninfo            │ Show current connection info
 \q                   │ Quit psql
```

### Power-User psql Tricks

```bash
# Execute a single command and exit
psql -c "SELECT version();"

# Execute a SQL file
psql -f migration.sql

# Output query results as CSV
psql -c "COPY (SELECT * FROM orders) TO STDOUT CSV HEADER" > orders.csv

# Connect with specific format
psql -At -c "SELECT datname FROM pg_database"
#  -A = unaligned output
#  -t = tuples only (no headers/footers)

# Use environment variables for connection (avoid passwords in commands)
export PGHOST=localhost
export PGPORT=5432
export PGUSER=myuser
export PGDATABASE=mydb
export PGPASSWORD=secret  # Or better: use .pgpass file
psql  # Connects automatically

# .pgpass file (recommended over env vars)
# ~/.pgpass format:  hostname:port:database:username:password
# chmod 600 ~/.pgpass
```

---

## 🎓 Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│  PostgreSQL Administration — The Cheat Sheet                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  👥 Roles = Users + Groups (everything is a role)            │
│  🔒 RLS = row-level access control (essential for SaaS)     │
│  📁 Schemas = namespaces for organizing objects              │
│  💾 pg_dump -Fc = best backup format (compressed, parallel)  │
│  ⏰ PITR = time-travel recovery (base backup + WAL archive)  │
│  🔄 pg_upgrade --link = fastest major version upgrade        │
│  🔐 scram-sha-256 + SSL + pg_hba.conf = security foundation │
│  📊 Monitor: cache hit %, index usage %, bloat, connections  │
│  🧹 Autovacuum handles most maintenance — but TUNE it       │
│  🧰 psql is incredibly powerful — learn the shortcuts        │
│                                                              │
│  Golden Rules:                                               │
│  1. Never use trust authentication in production             │
│  2. Always revoke PUBLIC privileges first, then grant         │
│  3. Test your backups by actually restoring them              │
│  4. Create restore points before scary operations            │
│  5. Monitor XID age — wraparound kills databases             │
└──────────────────────────────────────────────────────────────┘
```

---

> **Congratulations!** You've completed the PostgreSQL deep dive (Chapters 2E.1 → 2E.7).
> You now have the knowledge to install, configure, optimize, replicate, secure, and administer PostgreSQL at a production level.
>
> **Next up:** [2F.1 — SQLite — The Embedded Powerhouse](../07-SQLite/01-SQLite-Complete-Guide.md) 🟢⭐
> *The most deployed database in the world — in your phone, your browser, and everywhere.*
