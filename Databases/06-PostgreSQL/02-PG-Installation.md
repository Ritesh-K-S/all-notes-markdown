# 2E.2 — PostgreSQL Installation & Configuration 🟢

> **"You can learn PostgreSQL theory forever. Or you can install it and be writing queries in 10 minutes."**
> Let's get your hands dirty.

> **Level:** 🟢 Beginner
> **Time to Master:** ~1-2 hours
> **Prerequisites:** Chapter 2E.1 (PostgreSQL Architecture)

---

## 🎯 What You'll Master

```
✅ Install PostgreSQL on Windows, Linux, macOS, and Docker
✅ Understand the key configuration files (postgresql.conf, pg_hba.conf)
✅ Use psql — the command-line power tool every PG pro uses
✅ Use pgAdmin — the graphical admin tool
✅ Create databases, users, and schemas
✅ Tune essential settings for development AND production
✅ Explore the Extensions ecosystem — PostgreSQL's secret weapon
✅ Common first-timer mistakes and how to avoid them
```

---

## 🚀 1. Installation — Choose Your Path

### Option A: Windows Installation

```
Step 1: Download from https://www.postgresql.org/download/windows/
        → Use the EnterpriseDB installer (recommended)
        → Choose latest stable version (16.x or 17.x)

Step 2: Run the installer
        ┌─────────────────────────────────────────────┐
        │  ✅ PostgreSQL Server                        │
        │  ✅ pgAdmin 4 (graphical tool)               │
        │  ✅ Stack Builder (for extensions)            │
        │  ✅ Command Line Tools                       │
        └─────────────────────────────────────────────┘

Step 3: Set the superuser password
        → Username: postgres (default superuser)
        → Password: <choose a strong one>
        → ⚠️ REMEMBER THIS PASSWORD!

Step 4: Port → 5432 (default, keep it)

Step 5: Locale → Default locale (or C for binary sorting)
```

> 💡 **Add PostgreSQL to PATH:**
> Add `C:\Program Files\PostgreSQL\17\bin` to your system PATH
> → Now you can use `psql` from any terminal!

### Option B: Linux (Ubuntu/Debian)

```bash
# Add PostgreSQL official repo (gets you the LATEST version)
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Install
sudo apt update
sudo apt install postgresql-17 postgresql-contrib-17

# PostgreSQL starts automatically as a service
sudo systemctl status postgresql

# Switch to the postgres user and open psql
sudo -u postgres psql

# Inside psql — set a password for the postgres user
ALTER USER postgres PASSWORD 'your_secure_password';
```

### Option C: macOS

```bash
# Using Homebrew (recommended)
brew install postgresql@17

# Start the service
brew services start postgresql@17

# Connect
psql postgres
```

> 💡 **Alternatively:** Download [Postgres.app](https://postgresapp.com/) — a native macOS app. Just double-click and go.

### Option D: Docker (Best for Development & Learning) ⭐

```bash
# Pull and run PostgreSQL in one command
docker run --name my-postgres \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres:17

# Connect from your host machine
psql -h localhost -U admin -d myapp

# Or connect from inside the container
docker exec -it my-postgres psql -U admin -d myapp
```

**Docker Compose (production-like setup):**

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:17
    container_name: pg-dev
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret123
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Auto-run on first start
    restart: unless-stopped

volumes:
  pgdata:
```

```bash
docker compose up -d    # Start
docker compose down     # Stop
docker compose logs -f  # View logs
```

---

## 🔧 2. The Two Config Files You MUST Know

PostgreSQL has two critical configuration files. Master these and you master PostgreSQL administration.

### File 1: `postgresql.conf` — The Brain

**Location:**
- Linux: `/etc/postgresql/17/main/postgresql.conf`
- Windows: `C:\Program Files\PostgreSQL\17\data\postgresql.conf`
- Docker: `/var/lib/postgresql/data/postgresql.conf`

```bash
# Find it from psql
SHOW config_file;
```

**Essential Settings (with sane defaults for development):**

```ini
# ═══════════════════════════════════════════════════
# CONNECTIONS
# ═══════════════════════════════════════════════════
listen_addresses = 'localhost'      # '*' for remote access (careful!)
port = 5432
max_connections = 100               # Production: tune based on needs
                                    # (use connection pooling!)

# ═══════════════════════════════════════════════════
# MEMORY (THE BIG FOUR)
# ═══════════════════════════════════════════════════
shared_buffers = 128MB              # → Production: 25% of total RAM
                                    # 8GB RAM → 2GB shared_buffers

effective_cache_size = 4GB          # → Production: 50-75% of total RAM
                                    # Hint to planner (doesn't allocate memory)

work_mem = 4MB                      # → Production: 16-64MB
                                    # ⚠️ Per-operation, not per-query!

maintenance_work_mem = 64MB         # → Production: 256MB-1GB
                                    # Used by VACUUM, CREATE INDEX

# ═══════════════════════════════════════════════════
# WAL (Write-Ahead Logging)
# ═══════════════════════════════════════════════════
wal_level = replica                 # 'logical' if you need logical replication
max_wal_size = 1GB                  # → Production: 2-8GB
min_wal_size = 80MB
wal_compression = on                # Save I/O at cost of CPU

# ═══════════════════════════════════════════════════
# CHECKPOINT
# ═══════════════════════════════════════════════════
checkpoint_timeout = 5min
checkpoint_completion_target = 0.9  # Spread I/O over 90% of interval

# ═══════════════════════════════════════════════════
# LOGGING (ESSENTIAL for debugging)
# ═══════════════════════════════════════════════════
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_statement = 'ddl'              # none | ddl | mod | all
log_min_duration_statement = 1000  # Log queries slower than 1 second
log_line_prefix = '%t [%p] %u@%d '

# ═══════════════════════════════════════════════════
# AUTOVACUUM
# ═══════════════════════════════════════════════════
autovacuum = on                    # NEVER turn this off!
autovacuum_max_workers = 3
autovacuum_vacuum_scale_factor = 0.2
autovacuum_analyze_scale_factor = 0.1

# ═══════════════════════════════════════════════════
# LOCALE & ENCODING
# ═══════════════════════════════════════════════════
timezone = 'Asia/Kolkata'          # Set your timezone!
datestyle = 'iso, mdy'
default_text_search_config = 'pg_catalog.english'
```

> 💡 **How to apply changes:**
> ```sql
> -- Some settings can be changed without restart
> SELECT pg_reload_conf();   -- Reloads postgresql.conf
> 
> -- Some require restart (shared_buffers, max_connections, wal_level)
> -- Restart the PostgreSQL service
> ```

```sql
-- Check which settings need restart vs reload
SELECT name, setting, context 
FROM pg_settings 
WHERE context IN ('postmaster', 'sighup')
ORDER BY context, name;

-- context = 'postmaster' → needs restart
-- context = 'sighup'     → needs reload only
-- context = 'user'       → can SET per session
```

---

### File 2: `pg_hba.conf` — The Bouncer (Authentication)

**HBA = Host-Based Authentication.** This file controls WHO can connect, FROM WHERE, and HOW.

```bash
SHOW hba_file;    -- Find the file location
```

**Format:**

```
# TYPE    DATABASE    USER        ADDRESS           METHOD
# ────    ────────    ────        ───────           ──────

# Local connections (Unix socket)
local     all         postgres                      peer
local     all         all                           peer

# IPv4 local connections
host      all         all         127.0.0.1/32      scram-sha-256
host      all         all         0.0.0.0/0         scram-sha-256

# IPv6 local connections
host      all         all         ::1/128           scram-sha-256

# Allow specific user from specific IP
host      myapp_db    app_user    192.168.1.0/24    scram-sha-256

# Allow replication connections
host      replication rep_user    10.0.0.0/8        scram-sha-256
```

**Authentication Methods:**

| Method | Description | Use When |
|--------|-------------|----------|
| `trust` | No password needed! Anyone can connect. | ⚠️ NEVER in production! Development only. |
| `peer` | Uses OS username (must match PG username) | Local Unix connections |
| `scram-sha-256` | Password hash (most secure) ⭐ | Default for remote connections |
| `md5` | Password hash (legacy, less secure) | Legacy systems |
| `cert` | SSL client certificate | High-security environments |
| `ldap` | LDAP/Active Directory authentication | Enterprise environments |
| `reject` | Always reject — blacklist rule | Block specific IPs/users |

> ⚠️ **THE MOST COMMON BEGINNER MISTAKE:**
> ```
> "FATAL: Peer authentication failed for user 'postgres'"
> ```
> **Fix:** You're connecting via Unix socket with a different OS user. Either:
> ```bash
> # Option 1: Switch to postgres OS user
> sudo -u postgres psql
> 
> # Option 2: Connect via TCP (uses password auth)
> psql -h localhost -U postgres
> 
> # Option 3: Change pg_hba.conf from 'peer' to 'scram-sha-256' for local
> ```

> 💡 **After editing pg_hba.conf, reload:**
> ```sql
> SELECT pg_reload_conf();
> -- OR from command line:
> -- sudo systemctl reload postgresql
> ```

---

## 💻 3. `psql` — The Command-Line Power Tool

`psql` is PostgreSQL's interactive terminal. **Every serious DBA uses it daily.** It's far more powerful than most GUI tools.

### Connecting

```bash
# Basic connection
psql -U postgres                        # Connect as postgres user
psql -U admin -d myapp                  # Connect as admin to myapp database
psql -h 192.168.1.100 -U admin -d myapp # Connect to remote server
psql "postgresql://admin:pass@localhost:5432/myapp"  # Connection URI

# Connection parameters
# -h  host
# -p  port (default: 5432)
# -U  username
# -d  database
# -W  force password prompt
```

### The psql Meta-Commands (Backslash Commands) — Your Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────┐
│                    psql META-COMMANDS CHEAT SHEET                    │
├─────────────┬───────────────────────────────────────────────────────┤
│  COMMAND    │  WHAT IT DOES                                         │
├─────────────┼───────────────────────────────────────────────────────┤
│             │  📋 LISTING OBJECTS                                   │
│  \l         │  List all databases                                   │
│  \dt        │  List tables in current schema                        │
│  \dt+       │  List tables with sizes                               │
│  \dt *.     │  List tables in ALL schemas                           │
│  \di        │  List indexes                                         │
│  \dv        │  List views                                           │
│  \df        │  List functions                                       │
│  \dn        │  List schemas                                         │
│  \du        │  List users/roles                                     │
│  \ds        │  List sequences                                       │
│  \dx        │  List installed extensions                             │
│  \dp        │  List table privileges (permissions)                  │
├─────────────┼───────────────────────────────────────────────────────┤
│             │  🔍 DESCRIBING OBJECTS                                │
│  \d users   │  Describe table "users" (columns, types, indexes)    │
│  \d+ users  │  Detailed description (storage, stats, comments)     │
├─────────────┼───────────────────────────────────────────────────────┤
│             │  🔀 NAVIGATION                                       │
│  \c mydb    │  Connect to a different database                     │
│  \c mydb admin│ Connect as different user to different database    │
│  \cd /tmp   │  Change directory (for file operations)              │
├─────────────┼───────────────────────────────────────────────────────┤
│             │  📊 OUTPUT                                            │
│  \x         │  Toggle expanded display (vertical output)           │
│  \x auto    │  Auto-expand when rows are wide                      │
│  \timing    │  Toggle query execution timing                       │
│  \pset      │  Set output format (border, columns, etc.)           │
├─────────────┼───────────────────────────────────────────────────────┤
│             │  📁 FILE I/O                                         │
│  \i file.sql│  Execute SQL from a file                             │
│  \o out.txt │  Send output to a file                               │
│  \o         │  Stop sending to file (back to screen)               │
│  \copy      │  Import/export CSV (client-side COPY)                │
├─────────────┼───────────────────────────────────────────────────────┤
│             │  🛠️ UTILITIES                                        │
│  \e         │  Open query in $EDITOR (vim, nano, etc.)             │
│  \s         │  Show command history                                 │
│  \g         │  Execute last query again                            │
│  \watch 5   │  Re-run last query every 5 seconds                   │
│  \?         │  Help for psql commands                              │
│  \h SELECT  │  SQL syntax help for SELECT                          │
│  \q         │  Quit psql                                           │
│  \! ls      │  Run shell command                                   │
└─────────────┴───────────────────────────────────────────────────────┘
```

### psql Power Moves

```sql
-- Auto-expand output for wide tables
\x auto

-- Show query timing
\timing on

-- Execute a file
\i /path/to/schema.sql

-- Watch a query refresh every 2 seconds (like Linux watch command)
SELECT count(*) FROM orders WHERE status = 'pending';
\watch 2

-- Export query results to CSV
\copy (SELECT * FROM users WHERE country = 'India') TO '/tmp/indian_users.csv' WITH CSV HEADER

-- Import from CSV
\copy users(name, email, city) FROM '/tmp/users.csv' WITH CSV HEADER

-- Set variables
\set VERBOSE_ERRORS on
\set ON_ERROR_STOP on     -- Stop script on first error (great for migrations!)
```

### The `.psqlrc` File — Customize Your psql

Create `~/.psqlrc` (Linux/Mac) or `%APPDATA%\postgresql\psqlrc.conf` (Windows):

```sql
-- ~/.psqlrc — runs every time psql starts
\set ON_ERROR_ROLLBACK interactive
\set COMP_KEYWORD_CASE upper
\timing on
\x auto
\pset null '¤NULL¤'          -- Show NULL visually instead of blank
\set PROMPT1 '%n@%/%R%# '    -- Custom prompt: user@database=#

-- Useful shortcuts
\set activity 'SELECT pid, usename, state, query FROM pg_stat_activity WHERE state != \'idle\' ORDER BY query_start;'
\set sizes 'SELECT relname, pg_size_pretty(pg_total_relation_size(oid)) FROM pg_class WHERE relkind = \'r\' ORDER BY pg_total_relation_size(oid) DESC LIMIT 20;'
\set locks 'SELECT l.pid, l.mode, l.granted, a.query FROM pg_locks l JOIN pg_stat_activity a ON l.pid = a.pid WHERE NOT l.granted;'

-- Now you can run:  :activity   :sizes   :locks
```

---

## 🖥️ 4. pgAdmin — The Graphical Power Tool

**pgAdmin 4** is the official graphical management tool for PostgreSQL. It runs as a web app in your browser.

```
┌───────────────────────────────────────────────────────────────┐
│                      pgAdmin 4                                │
│                                                               │
│  ┌─────────────┐  ┌───────────────────────────────────────┐ │
│  │  Object      │  │  Query Tool                           │ │
│  │  Browser     │  │  ┌─────────────────────────────────┐ │ │
│  │              │  │  │ SELECT * FROM users              │ │ │
│  │  📁 Servers  │  │  │ WHERE country = 'India'          │ │ │
│  │   📁 mydb   │  │  │ ORDER BY name;                   │ │ │
│  │    📁 Schemas│  │  └─────────────────────────────────┘ │ │
│  │     📁 public│  │  ▶ Run  💾 Save  📊 Explain          │ │
│  │      📋 users│  │                                       │ │
│  │      📋 orders│  │  ┌─────────────────────────────────┐ │ │
│  │     📁 func  │  │  │ id │ name    │ city  │ country  │ │ │
│  │              │  │  │  1 │ Ritesh  │ Delhi │ India    │ │ │
│  │              │  │  │  6 │ Priya   │ Mumbai│ India    │ │ │
│  └─────────────┘  │  └─────────────────────────────────┘ │ │
│                    └───────────────────────────────────────┘ │
│                                                               │
│  Features: Query tool, ERD designer, Dashboard, Backup/       │
│  Restore GUI, Import/Export, Server monitoring                 │
└───────────────────────────────────────────────────────────────┘
```

**Other Popular GUI Tools:**

| Tool | Platform | Free? | Best For |
|------|----------|-------|----------|
| **pgAdmin 4** | All | ✅ | Official tool, full admin |
| **DBeaver** | All | ✅ | Multi-database support |
| **DataGrip** | All | ❌ (paid) | JetBrains fans, best autocomplete |
| **Azure Data Studio** | All | ✅ | PostgreSQL + SQL Server |
| **Postico** | macOS | ❌ (paid) | Beautiful, lightweight |
| **TablePlus** | All | Freemium | Clean UI, multiple databases |

---

## 🛠️ 5. First Steps — Your First Database

### Create a User (Role)

```sql
-- In psql as the postgres superuser:

-- Create a regular user (role)
CREATE USER app_user WITH PASSWORD 'secure_password_123';

-- Create a superuser (for admin tasks only!)
CREATE USER admin_user WITH SUPERUSER PASSWORD 'admin_pass_456';

-- Create a role with specific privileges
CREATE ROLE readonly_role;
GRANT CONNECT ON DATABASE myapp TO readonly_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly_role;

-- Assign role to user
GRANT readonly_role TO app_user;
```

> 💡 **In PostgreSQL, USER = ROLE WITH LOGIN.** They're the same thing internally.
> ```sql
> CREATE USER bob WITH PASSWORD 'x';
> -- is identical to:
> CREATE ROLE bob WITH LOGIN PASSWORD 'x';
> ```

### Create a Database

```sql
-- Create database
CREATE DATABASE myapp 
  OWNER app_user 
  ENCODING 'UTF8' 
  LC_COLLATE 'en_US.UTF-8'
  LC_CTYPE 'en_US.UTF-8'
  TEMPLATE template0;

-- Connect to it
\c myapp

-- Create a schema (namespace for tables)
CREATE SCHEMA app;

-- Set the search path (so you don't need schema prefix)
ALTER DATABASE myapp SET search_path TO app, public;
-- Or per user:
ALTER ROLE app_user SET search_path TO app, public;
```

### Create Your First Table

```sql
-- Connect to your database
\c myapp

-- Create a users table
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,           -- Auto-incrementing integer
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    full_name   VARCHAR(100),
    created_at  TIMESTAMPTZ DEFAULT NOW(),    -- Timezone-aware timestamp!
    is_active   BOOLEAN DEFAULT true,
    metadata    JSONB DEFAULT '{}'::jsonb      -- Yes, JSONB is a first-class type!
);

-- Insert some data
INSERT INTO users (username, email, full_name) VALUES
  ('ritesh', 'ritesh@example.com', 'Ritesh Singh'),
  ('sarah', 'sarah@example.com', 'Sarah Connor'),
  ('akira', 'akira@example.com', 'Akira Tanaka');

-- Query it
SELECT * FROM users;

-- Check table structure
\d users
\d+ users
```

### `SERIAL` vs `IDENTITY` — The Modern Way

```sql
-- OLD way (still works, very common)
CREATE TABLE old_way (
    id SERIAL PRIMARY KEY    -- Creates a sequence behind the scenes
);

-- NEW way (SQL standard, PostgreSQL 10+) ⭐
CREATE TABLE new_way (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- IDENTITY with options
CREATE TABLE better_way (
    id INT GENERATED ALWAYS AS IDENTITY (START 1000 INCREMENT 1) PRIMARY KEY
);
```

| Feature | `SERIAL` | `GENERATED ALWAYS AS IDENTITY` |
|---------|----------|-------------------------------|
| SQL Standard | ❌ PostgreSQL-specific | ✅ SQL:2003 standard |
| Can be overridden | Anyone can INSERT any value | Must use `OVERRIDING SYSTEM VALUE` |
| Sequence ownership | Separate sequence object | Tied to column |
| Recommendation | Legacy — still fine | ⭐ Preferred for new projects |

---

## 🧩 6. The Extensions Ecosystem — PostgreSQL's Superpower

**Extensions** are what make PostgreSQL truly unique. They let you add entire new capabilities without modifying the core.

```sql
-- List available extensions
SELECT * FROM pg_available_extensions ORDER BY name;

-- Install an extension
CREATE EXTENSION IF NOT EXISTS extension_name;

-- List installed extensions
\dx
-- or
SELECT * FROM pg_extension;

-- Remove an extension
DROP EXTENSION extension_name;
```

### Must-Know Extensions

```
┌───────────────────────────────────────────────────────────────────────┐
│                    ESSENTIAL EXTENSIONS                                │
├──────────────────────┬────────────────────────────────────────────────┤
│  Extension           │  What It Does                                  │
├──────────────────────┼────────────────────────────────────────────────┤
│                      │                                                │
│  🔥 pg_stat_statements│ Track execution statistics of all SQL         │
│                      │  queries. THE #1 performance tool.             │
│                      │  → See slowest queries, most called queries    │
│                      │                                                │
│  🔥 pgcrypto         │ Cryptographic functions                        │
│                      │  → gen_random_uuid(), crypt(), digest()        │
│                      │                                                │
│  🔥 uuid-ossp        │ UUID generation functions                      │
│                      │  → uuid_generate_v4()                          │
│                      │  💡 In PG 13+, use gen_random_uuid() instead   │
│                      │                                                │
│  🔥 pg_trgm          │ Trigram-based text similarity & search         │
│                      │  → Fuzzy matching, LIKE/ILIKE index support    │
│                      │                                                │
│  ⭐ hstore           │ Key-value store in a column                    │
│                      │  → Simpler alternative to JSONB for flat data  │
│                      │                                                │
│  ⭐ tablefunc        │ Crosstab (pivot) functions                     │
│                      │  → Pivot table functionality                    │
│                      │                                                │
│  ⭐ btree_gist       │ GiST operator classes for B-tree types         │
│                      │  → Needed for exclusion constraints            │
│                      │                                                │
│  ⭐ citext           │ Case-insensitive text type                     │
│                      │  → email CITEXT — no more LOWER() everywhere  │
│                      │                                                │
│  🌍 PostGIS          │ Geospatial support (external)                  │
│                      │  → GPS coordinates, distance, geometry, maps   │
│                      │                                                │
│  🤖 pgvector         │ Vector similarity search (external)            │
│                      │  → AI/ML embeddings, RAG, nearest neighbors   │
│                      │                                                │
│  ⏰ pg_cron          │ Job scheduler inside PostgreSQL (external)     │
│                      │  → Schedule SQL jobs like cron                 │
│                      │                                                │
│  📊 TimescaleDB      │ Time-series superpowers (external)             │
│                      │  → Hypertables, compression, continuous aggs   │
│                      │                                                │
│  🏙️  Citus           │ Distributed PostgreSQL (external)              │
│                      │  → Shard tables across multiple nodes          │
│                      │                                                │
└──────────────────────┴────────────────────────────────────────────────┘
```

### Quick Extension Demo

```sql
-- Enable pg_stat_statements (requires server config change + restart)
-- In postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- Then restart, then:
CREATE EXTENSION pg_stat_statements;

-- See top 10 slowest queries
SELECT 
    query,
    calls,
    mean_exec_time::numeric(10,2) AS avg_ms,
    total_exec_time::numeric(10,2) AS total_ms,
    rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- or uuid-ossp

-- Use UUIDs as primary keys
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    price NUMERIC(10,2)
);

INSERT INTO products (name, price) VALUES ('Widget', 29.99);
SELECT * FROM products;
-- id: 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11' (auto-generated!)

-- Enable case-insensitive text
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TABLE accounts (
    email CITEXT UNIQUE NOT NULL    -- 'John@Email.com' = 'john@email.com'
);

INSERT INTO accounts VALUES ('John@Email.com');
INSERT INTO accounts VALUES ('john@email.com');  -- ERROR: duplicate! ✅
```

---

## 📁 7. PostgreSQL Directory Structure

Understanding what's where helps you debug problems fast.

```
$PGDATA (e.g., /var/lib/postgresql/17/main/)
├── postgresql.conf          ← Main configuration
├── pg_hba.conf              ← Authentication rules
├── pg_ident.conf            ← Username mapping
├── PG_VERSION               ← PostgreSQL major version
├── postmaster.pid           ← PID file (running server)
│
├── base/                    ← Database files (one subdir per database)
│   ├── 1/                   ← template1
│   ├── 13356/               ← template0
│   └── 16384/               ← your_database (OID-named)
│       ├── 16389            ← table file (OID-named)
│       ├── 16389_fsm        ← free space map
│       └── 16389_vm         ← visibility map
│
├── global/                  ← Cluster-wide tables (pg_database, pg_auth)
│
├── pg_wal/                  ← WAL files (16MB each)
│   ├── 000000010000000000000001
│   ├── 000000010000000000000002
│   └── ...
│
├── pg_xact/                 ← Transaction commit status (CLOG)
├── pg_stat/                 ← Statistics files
├── pg_stat_tmp/             ← Temporary statistics
├── pg_tblspc/               ← Symbolic links to tablespaces
├── pg_twophase/             ← Prepared transactions
├── pg_multixact/            ← Multi-transaction status
├── pg_subtrans/             ← Sub-transaction status
├── pg_notify/               ← LISTEN/NOTIFY data
├── pg_serial/               ← Serializable transaction info
├── pg_snapshots/            ← Exported snapshots
├── pg_logical/              ← Logical replication data
└── log/                     ← Server logs (if logging_collector = on)
```

> 💡 **Never manually modify files in `$PGDATA`!** Use SQL commands or PostgreSQL tools instead.

---

## ⚡ 8. Production Configuration Quick Start

Here's a **copy-paste production config** for a server with **16GB RAM**:

```ini
# ═══════════════════════════════════════════════
#  PRODUCTION postgresql.conf (16GB RAM server)
# ═══════════════════════════════════════════════

# ─── Connections ───
max_connections = 200
superuser_reserved_connections = 3

# ─── Memory ───
shared_buffers = 4GB               # 25% of 16GB
effective_cache_size = 12GB        # 75% of 16GB
work_mem = 32MB                    # Careful — per operation!
maintenance_work_mem = 512MB       # For VACUUM, indexes
huge_pages = try                   # Use huge pages if available

# ─── WAL ───
wal_level = replica
wal_compression = on
max_wal_size = 4GB
min_wal_size = 1GB
wal_buffers = 64MB

# ─── Checkpoint ───
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

# ─── Planner ───
random_page_cost = 1.1             # SSD storage (default 4.0 is for HDD!)
effective_io_concurrency = 200     # SSD (default 1 is for HDD)
default_statistics_target = 200    # Better query plans (default 100)

# ─── Parallel Queries ───
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4

# ─── Logging ───
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%a.log'  # Day-of-week rotation
log_min_duration_statement = 500    # Log queries > 500ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0                  # Log all temp file usage

# ─── Autovacuum (aggressive) ───
autovacuum = on
autovacuum_max_workers = 4
autovacuum_vacuum_scale_factor = 0.05    # 5% instead of 20%
autovacuum_analyze_scale_factor = 0.025  # 2.5% instead of 10%
autovacuum_vacuum_cost_delay = 2ms
```

### The Settings Scaling Formula

```
┌────────────────────────┬──────────┬──────────┬──────────┬──────────┐
│  Setting               │   8GB    │   16GB   │   32GB   │   64GB   │
├────────────────────────┼──────────┼──────────┼──────────┼──────────┤
│  shared_buffers        │   2GB    │   4GB    │   8GB    │  16GB    │
│  effective_cache_size  │   6GB    │  12GB    │  24GB    │  48GB    │
│  work_mem              │  16MB    │  32MB    │  64MB    │ 128MB    │
│  maintenance_work_mem  │ 256MB    │ 512MB    │   1GB    │   2GB    │
│  max_wal_size          │   2GB    │   4GB    │   8GB    │   8GB    │
└────────────────────────┴──────────┴──────────┴──────────┴──────────┘
```

> 💡 **Use PGTune for automatic recommendations:** https://pgtune.leopard.in.ua/
> Enter your server specs → get optimized postgresql.conf values.

---

## ❌ 9. Common Mistakes & How to Avoid Them

```
┌─────────────────────────────────────────────────────────────────────┐
│   MISTAKE                          │  FIX                           │
├────────────────────────────────────┼────────────────────────────────┤
│ Using trust auth in production     │ Use scram-sha-256              │
│ Not setting a superuser password   │ ALTER USER postgres PASSWORD   │
│ shared_buffers at default 128MB    │ Set to 25% of RAM              │
│ random_page_cost = 4.0 on SSD      │ Set to 1.1 for SSD            │
│ Not enabling pg_stat_statements    │ Add to shared_preload_libraries│
│ Forgetting to ANALYZE after load   │ Run ANALYZE; after bulk insert │
│ Using SERIAL instead of IDENTITY   │ Use GENERATED ALWAYS           │
│ Not using connection pooling       │ Deploy PgBouncer                │
│ log_statement = 'all' in prod      │ Use log_min_duration_statement │
│ Running VACUUM FULL routinely      │ Let autovacuum handle it       │
│ Connecting as superuser from app   │ Create app-specific role       │
└────────────────────────────────────┴────────────────────────────────┘
```

---

## 📝 Chapter Summary — Key Takeaways

```
┌────────────────────────────────────────────────────────────────────┐
│  🧠 REMEMBER THESE:                                               │
│                                                                     │
│  1. Install via Docker for learning, native for production         │
│                                                                     │
│  2. TWO config files matter most:                                  │
│     • postgresql.conf → performance & behavior                     │
│     • pg_hba.conf → who can connect & how                          │
│                                                                     │
│  3. psql is your best friend — learn the backslash commands        │
│     → \dt (tables), \d+ table (describe), \x (expand), \timing    │
│                                                                     │
│  4. Memory tuning: shared_buffers=25% RAM, work_mem=careful!       │
│                                                                     │
│  5. Use SSD settings: random_page_cost=1.1, effective_io_conc=200  │
│                                                                     │
│  6. Extensions are PostgreSQL's superpower:                        │
│     pg_stat_statements, pgcrypto, pg_trgm, PostGIS, pgvector       │
│                                                                     │
│  7. Use IDENTITY columns instead of SERIAL for new projects        │
│                                                                     │
│  8. Never use trust auth or superuser in production apps           │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2E.3 — PostgreSQL Advanced Features](./03-PG-Advanced-Features.md) 🟡⭐🔥
> *JSONB, Arrays, Full-Text Search, PostGIS — the features that make PostgreSQL irresistible.*

---

[← Back to Index](../INDEX.md)
