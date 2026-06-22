# 2D.2 — MySQL Installation & Configuration 🟢

> **"A poorly configured MySQL is like a Ferrari in first gear — all that power, completely wasted."**

---

## 📌 What You'll Master in This Chapter

- MySQL **editions** — Community vs Enterprise vs Cloud (what to pick and when)
- **Step-by-step installation** on Windows, Linux (Ubuntu/CentOS), macOS, and Docker
- **MySQL Workbench** — your visual command center
- **mysql CLI** — the terminal powerhouse every DBA must master
- **my.cnf / my.ini** — the configuration file that controls everything
- **Critical tuning parameters** that separate a toy setup from production-grade
- **Post-installation security** hardening

---

## 🔥 1. MySQL Editions — Which One Do You Need?

```
┌──────────────────────────────────────────────────────────────────┐
│                     MYSQL EDITIONS                                │
├────────────────┬─────────────────────────────────────────────────┤
│                │                                                  │
│  COMMUNITY     │  ✅ FREE (GPL license)                           │
│  EDITION       │  ✅ All core features (InnoDB, replication, etc.)│
│                │  ✅ Perfect for development and most production   │
│                │  ❌ No official Oracle support                    │
│                │  ❌ No enterprise plugins (audit, firewall, etc.) │
│                │  📥 Download: dev.mysql.com/downloads             │
│                │                                                  │
├────────────────┼─────────────────────────────────────────────────┤
│                │                                                  │
│  ENTERPRISE    │  💰 Paid (Oracle commercial license)             │
│  EDITION       │  ✅ Everything in Community PLUS:                │
│                │  ✅ Thread Pool (high-concurrency performance)    │
│                │  ✅ Enterprise Audit Plugin                       │
│                │  ✅ Enterprise Firewall                           │
│                │  ✅ Enterprise Backup (hot online backup)         │
│                │  ✅ Enterprise Monitor                            │
│                │  ✅ Enterprise Encryption (TDE)                   │
│                │  ✅ 24/7 Oracle Support                           │
│                │                                                  │
├────────────────┼─────────────────────────────────────────────────┤
│                │                                                  │
│  CLOUD /       │  ☁️ Managed MySQL services:                      │
│  MANAGED       │  • Amazon RDS for MySQL / Aurora MySQL            │
│                │  • Azure Database for MySQL                       │
│                │  • Google Cloud SQL for MySQL                     │
│                │  • DigitalOcean Managed MySQL                     │
│                │  ✅ Auto-backup, patching, scaling                │
│                │  ✅ High availability built-in                    │
│                │  ❌ Less control over config                      │
│                │  ❌ Vendor lock-in risk                           │
│                │                                                  │
├────────────────┼─────────────────────────────────────────────────┤
│                │                                                  │
│  MariaDB       │  ✅ FREE fork of MySQL (by original creator)     │
│  (Alternative) │  ✅ Drop-in replacement for MySQL 5.x            │
│                │  ✅ Some advanced features (Columnstore, Spider)  │
│                │  ⚠️ Diverging from MySQL 8.0 compatibility       │
│                │                                                  │
└────────────────┴─────────────────────────────────────────────────┘
```

> 💡 **Pro Tip:** For learning and most production workloads, **Community Edition** is all you need. Only consider Enterprise if you need specific plugins (audit, firewall) or Oracle support contracts.

---

## 🔥 2. Installation — Every Platform

### 🪟 Windows Installation

```
Step 1: Download MySQL Installer from dev.mysql.com/downloads/installer/
        → Choose "mysql-installer-community-x.x.x.msi"

Step 2: Run the installer → Choose Setup Type:
        ┌──────────────────────────────────────────────┐
        │  Developer Default  ← RECOMMENDED            │
        │  (Server + Workbench + Shell + Connectors)   │
        │                                              │
        │  Server Only       ← Minimal                 │
        │  Full              ← Everything               │
        │  Custom            ← Pick what you want       │
        └──────────────────────────────────────────────┘

Step 3: Configuration:
        → Standalone MySQL Server
        → Port: 3306 (default)
        → Authentication: Use Strong Password Encryption (caching_sha2_password)
        → Set ROOT password (REMEMBER THIS!)
        → Optionally create a user account

Step 4: Windows Service:
        → Configure as Windows Service ✅
        → Start at System Startup ✅
        → Service name: MySQL80

Step 5: Apply Configuration → Finish!
```

```powershell
# Verify installation (open PowerShell or cmd)
mysql --version
# mysql  Ver 8.0.xx for Win64 on x86_64

# Connect to MySQL
mysql -u root -p
# Enter your root password
```

### 🐧 Linux (Ubuntu/Debian) Installation

```bash
# Method 1: APT (recommended for Ubuntu 20.04+)
# ───────────────────────────────────────────────
sudo apt update
sudo apt install mysql-server -y

# Secure the installation (VERY IMPORTANT!)
sudo mysql_secure_installation
# → Set root password
# → Remove anonymous users: YES
# → Disallow root remote login: YES
# → Remove test database: YES
# → Reload privileges: YES

# Start & enable MySQL service
sudo systemctl start mysql
sudo systemctl enable mysql
sudo systemctl status mysql

# Connect
sudo mysql -u root -p
```

```bash
# Method 2: Official MySQL APT Repository (for latest version)
# ────────────────────────────────────────────────────────────
wget https://dev.mysql.com/get/mysql-apt-config_0.8.29-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb
# → Select MySQL 8.0 (or 8.4 LTS)
sudo apt update
sudo apt install mysql-server -y
```

### 🐧 Linux (RHEL/CentOS/Rocky) Installation

```bash
# Add MySQL Yum Repository
sudo yum install https://dev.mysql.com/get/mysql80-community-release-el8-9.noarch.rpm -y

# Install MySQL Server
sudo yum install mysql-server -y

# Start & enable
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Get temporary root password (MySQL generates one!)
sudo grep 'temporary password' /var/log/mysqld.log
# → Something like: A temporary password is generated for root@localhost: xK3!abc123

# First login (MUST change password immediately)
mysql -u root -p
# Enter the temporary password, then:
ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewStrongPassword123!';
```

### 🍎 macOS Installation

```bash
# Method 1: Homebrew (RECOMMENDED)
brew install mysql

# Start MySQL
brew services start mysql

# Secure installation
mysql_secure_installation

# Connect
mysql -u root -p

# ──────────────────────────────
# Method 2: DMG Installer
# Download from dev.mysql.com/downloads/mysql/
# Double-click the .dmg → Follow wizard
# MySQL goes to /usr/local/mysql/
# Start via System Preferences → MySQL panel
```

### 🐳 Docker Installation (Best for Development!)

```bash
# Pull official MySQL image
docker pull mysql:8.0

# Run MySQL container
docker run --name mysql-dev \
    -e MYSQL_ROOT_PASSWORD=rootpass123 \
    -e MYSQL_DATABASE=myapp \
    -e MYSQL_USER=devuser \
    -e MYSQL_PASSWORD=devpass123 \
    -p 3306:3306 \
    -v mysql-data:/var/lib/mysql \
    -d mysql:8.0

# Connect from host
mysql -h 127.0.0.1 -P 3306 -u root -prootpass123

# Or connect from inside the container
docker exec -it mysql-dev mysql -u root -prootpass123

# Docker Compose (recommended for projects)
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-dev
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpass123
      MYSQL_DATABASE: myapp
      MYSQL_USER: devuser
      MYSQL_PASSWORD: devpass123
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./my-custom.cnf:/etc/mysql/conf.d/custom.cnf  # Custom config
    command: --default-authentication-plugin=caching_sha2_password

volumes:
  mysql-data:
```

> 💡 **Why Docker for Development?** Instant setup, disposable, version pinning, consistent across team members. Use Docker locally, managed services (RDS/Cloud SQL) in production.

---

## 🔥 3. Post-Installation — First Steps

### Verify Everything Works

```sql
-- Connect to MySQL
mysql -u root -p

-- Check version
SELECT VERSION();
-- 8.0.36

-- Check current user
SELECT USER(), CURRENT_USER();

-- Check default storage engine
SHOW VARIABLES LIKE 'default_storage_engine';
-- InnoDB ✅

-- List databases
SHOW DATABASES;
/*
+--------------------+
| Database           |
+--------------------+
| information_schema |   ← Metadata about all databases/tables/columns
| mysql              |   ← MySQL system database (users, privileges)
| performance_schema |   ← Performance monitoring data
| sys                |   ← Easy-to-read views on performance_schema
+--------------------+
*/

-- Create your first database
CREATE DATABASE techmart CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE techmart;
```

### Understanding MySQL System Databases

```
┌────────────────────────────────────────────────────────────────┐
│  SYSTEM DATABASES (Never delete these!)                        │
├────────────────┬───────────────────────────────────────────────┤
│                │                                               │
│  mysql         │  THE system database. Contains:               │
│                │  • user table (accounts & privileges)          │
│                │  • db table (database-level privileges)        │
│                │  • tables_priv, columns_priv                   │
│                │  • time_zone tables                            │
│                │  • slow_log, general_log (if table logging)    │
│                │                                               │
│  information   │  READ-ONLY metadata catalog:                  │
│  _schema       │  • TABLES, COLUMNS, STATISTICS                │
│                │  • KEY_COLUMN_USAGE, REFERENTIAL_CONSTRAINTS   │
│                │  • PROCESSLIST, ENGINES, CHARACTER_SETS        │
│                │  Think of it as "SHOW commands in table form"  │
│                │                                               │
│  performance   │  Performance monitoring:                       │
│  _schema       │  • Wait events, mutex, stages                  │
│                │  • Statement history, memory usage              │
│                │  • Low overhead (instruments are toggleable)    │
│                │                                               │
│  sys           │  Human-friendly views on performance_schema:   │
│                │  • sys.session (who's connected?)               │
│                │  • sys.schema_table_statistics                  │
│                │  • sys.statements_with_full_table_scans         │
│                │  Great for quick diagnostics!                   │
│                │                                               │
└────────────────┴───────────────────────────────────────────────┘
```

---

## 🔥 4. MySQL Client Tools — Your Arsenal

### 4.1 mysql CLI — The Terminal Powerhouse

The `mysql` command-line client is the most powerful and most used tool for MySQL.

```bash
# Basic connection
mysql -u root -p

# Connect to specific database
mysql -u root -p techmart

# Connect to remote server
mysql -h 192.168.1.100 -P 3306 -u admin -p

# Execute a single query (great for scripts!)
mysql -u root -p -e "SHOW DATABASES;"

# Execute SQL file
mysql -u root -p techmart < /path/to/schema.sql

# Execute with output in a format
mysql -u root -p -e "SELECT * FROM users" --table      # Pretty table
mysql -u root -p -e "SELECT * FROM users" --batch       # Tab-separated
mysql -u root -p -e "SELECT * FROM users" --xml         # XML output
mysql -u root -p -e "SELECT * FROM users" --json        # JSON (8.0.27+)
```

### Essential mysql CLI Commands (Inside the Shell)

```sql
-- Navigation
SHOW DATABASES;
USE database_name;
SHOW TABLES;
DESCRIBE table_name;          -- or DESC table_name
SHOW CREATE TABLE table_name;  -- Full DDL with constraints

-- System Info
STATUS;                       -- Connection info, uptime, threads
SHOW PROCESSLIST;             -- What's running right now
SHOW VARIABLES LIKE '%timeout%';  -- Configuration variables
SHOW STATUS LIKE 'Threads%';      -- Runtime status counters

-- Formatting
\G     -- Vertical output (great for wide results)
       -- Example: SELECT * FROM users LIMIT 1\G

-- Help
HELP;
HELP SELECT;
HELP SHOW;

-- System
\! ls                         -- Run OS commands (Linux/Mac)
\! dir                        -- Run OS commands (Windows)
system cls                    -- Clear screen (Windows)
system clear                  -- Clear screen (Linux/Mac)

-- Edit
\e                            -- Open query in $EDITOR
EDIT                          -- Same

-- Source (execute SQL file)
SOURCE /path/to/file.sql;
\. /path/to/file.sql

-- Exit
EXIT;
QUIT;
\q
```

### 4.2 MySQL Workbench — The Visual Powerhouse

```
┌────────────────────────────────────────────────────────────────┐
│  MYSQL WORKBENCH — What It Can Do                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  📝 SQL Editor                                                 │
│     • Write & execute queries with syntax highlighting         │
│     • Auto-complete for tables, columns, functions             │
│     • Multiple tabs, query history                              │
│     • Visual EXPLAIN (execution plan as diagram)               │
│                                                                │
│  🎨 Visual Database Design                                     │
│     • Create ER diagrams (drag & drop!)                        │
│     • Forward engineer: Diagram → SQL → Database               │
│     • Reverse engineer: Database → ER Diagram                  │
│     • Export diagrams as PNG/PDF                                │
│                                                                │
│  🔧 Server Administration                                      │
│     • User & privilege management (GUI)                        │
│     • Server status & performance dashboard                     │
│     • Startup/shutdown                                          │
│     • Configuration file editor                                 │
│                                                                │
│  📦 Data Migration                                              │
│     • Migrate from SQL Server, PostgreSQL, etc. to MySQL       │
│     • Import/Export CSV, JSON, SQL dump                         │
│                                                                │
│  📥 Download: dev.mysql.com/downloads/workbench/                │
│  💰 Price: FREE (Community Edition)                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 4.3 MySQL Shell (mysqlsh) — The Modern CLI

```bash
# MySQL Shell — the next-generation CLI (supports SQL, JS, Python!)
mysqlsh root@localhost

# Switch modes
\sql              -- SQL mode
\js               -- JavaScript mode
\py               -- Python mode

# SQL mode example
\sql
SELECT * FROM techmart.users;

# JavaScript mode example
\js
var result = session.runSql("SELECT * FROM techmart.users");
print(result.fetchAll());

# Python mode example
\py
result = session.run_sql("SELECT * FROM techmart.users")
print(result.fetch_all())

# Great for:
# • InnoDB Cluster administration
# • JSON document operations (X DevAPI)
# • Scripting complex admin tasks
```

### 4.4 Other Useful Tools

| Tool | Purpose | Install |
|------|---------|---------|
| **DBeaver** | Universal GUI client (free, supports ALL databases) | dbeaver.io |
| **HeidiSQL** | Lightweight Windows GUI (fast, free) | heidisql.com |
| **DataGrip** | JetBrains IDE for databases (paid, powerful) | jetbrains.com |
| **phpMyAdmin** | Web-based GUI (popular with PHP/WordPress devs) | phpmyadmin.net |
| **Adminer** | Single-file web GUI (lightweight alternative to phpMyAdmin) | adminer.org |
| **mycli** | Enhanced CLI with auto-complete & syntax highlighting | mycli.net |
| **pt-query-digest** | Slow query analysis tool (Percona Toolkit) | percona.com |

> 💡 **My Recommendation:** Use **mysql CLI** for daily work (speed!), **MySQL Workbench** for visual modeling and admin, and **DBeaver** as a universal backup GUI.

---

## 🔥 5. Configuration — my.cnf / my.ini Deep Dive

This is where the magic happens. The MySQL configuration file controls every aspect of server behavior.

### Where to Find It

```
┌───────────────┬──────────────────────────────────────────────┐
│ Platform      │ Config File Location                          │
├───────────────┼──────────────────────────────────────────────┤
│ Linux         │ /etc/mysql/my.cnf                             │
│               │ /etc/my.cnf                                   │
│               │ ~/.my.cnf (user-specific)                     │
│ Windows       │ C:\ProgramData\MySQL\MySQL Server 8.0\my.ini │
│ macOS (Brew)  │ /usr/local/etc/my.cnf  (or /opt/homebrew/etc)│
│ Docker        │ /etc/mysql/conf.d/*.cnf                       │
└───────────────┴──────────────────────────────────────────────┘

# Check which files MySQL reads (and in what order!)
mysql --help | grep -A 10 "Default options"
# Or on Windows:
mysqld --help --verbose 2>&1 | Select-String "Default options" -Context 0,10
```

### Configuration File Structure

```ini
# ─────────────────────────────────────────────────────
# my.cnf — MySQL Configuration File
# ─────────────────────────────────────────────────────

[client]
# Settings for ALL client programs (mysql, mysqldump, etc.)
port = 3306
socket = /var/run/mysqld/mysqld.sock
default-character-set = utf8mb4

[mysql]
# Settings specific to the mysql CLI client
prompt = "mysql [\d]> "        # Shows database name in prompt!
auto-rehash                     # Enable tab-completion

[mysqld]
# ──────────────────────────────────────
# SERVER SETTINGS (the important stuff)
# ──────────────────────────────────────
user = mysql
port = 3306
bind-address = 0.0.0.0         # 0.0.0.0 = listen on all interfaces
                                # 127.0.0.1 = local only (more secure)
datadir = /var/lib/mysql
socket = /var/run/mysqld/mysqld.sock
pid-file = /var/run/mysqld/mysqld.pid

[mysqldump]
# Settings for mysqldump
quick                           # Don't buffer entire result in memory
max_allowed_packet = 256M
single-transaction              # Consistent backup without locking
```

> ⭐ **Key Insight:** Settings in `[mysqld]` affect the server. Settings in `[client]` affect all client tools. Settings in `[mysql]` affect only the mysql CLI. Don't mix them up!

---

## 🔥 6. Critical Tuning Parameters — The Essential Ones

### 6.1 Memory & Buffer Settings

```ini
[mysqld]
# ═══════════════════════════════════════════
# INNODB SETTINGS (Most important section!)
# ═══════════════════════════════════════════

# BUFFER POOL — #1 most impactful setting
# Rule: 70-80% of total RAM on dedicated servers
innodb_buffer_pool_size = 12G             # ⭐ MOST IMPORTANT SETTING
innodb_buffer_pool_instances = 8          # Parallel pools (for > 1GB)

# REDO LOG
innodb_redo_log_capacity = 4G             # MySQL 8.0.30+ (total redo log size)
# innodb_log_file_size = 1G              # Pre-8.0.30 (each file's size)
# innodb_log_files_in_group = 2          # Pre-8.0.30 (number of files)
innodb_flush_log_at_trx_commit = 1        # 1=SAFEST, 2=faster, 0=fastest
innodb_log_buffer_size = 64M              # Buffer before writing to redo log

# I/O TUNING
innodb_io_capacity = 2000                 # IOPS your disk can handle (HDD: 200, SSD: 2000+)
innodb_io_capacity_max = 4000             # Max IOPS during flushing
innodb_flush_method = O_DIRECT            # Linux: bypass OS cache (avoids double caching)

# CONCURRENCY
innodb_thread_concurrency = 0             # 0 = let InnoDB decide (usually best)

# FILE PER TABLE
innodb_file_per_table = ON                # Each table gets its own .ibd file (default)

# DOUBLEWRITE
innodb_doublewrite = ON                   # Default. Only disable on ZFS/battery-backed cache


# ═══════════════════════════════════════════
# GENERAL SERVER SETTINGS
# ═══════════════════════════════════════════

# CHARACTER SET (ALWAYS use utf8mb4!)
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
# ⚠️ utf8 in MySQL is actually utf8mb3 (3 bytes, no emoji!)
# utf8mb4 is REAL UTF-8 (4 bytes, supports emoji 😀)

# CONNECTIONS
max_connections = 500                     # Default 151 (too low for production)
wait_timeout = 28800                      # Close idle connections after 8 hours
interactive_timeout = 28800
max_allowed_packet = 64M                  # Max size of a single packet/query

# TEMPORARY TABLES
tmp_table_size = 256M                     # Max in-memory temp table
max_heap_table_size = 256M                # Must match tmp_table_size

# TABLE CACHE
table_open_cache = 4000                   # Cached open table handles
table_definition_cache = 2000             # Cached .frm files

# QUERY EXECUTION
sort_buffer_size = 4M                     # Per-connection sort buffer
join_buffer_size = 4M                     # Per-connection join buffer
read_buffer_size = 2M                     # Sequential scan buffer
read_rnd_buffer_size = 4M                 # Random read buffer
```

### 6.2 Logging Settings

```ini
[mysqld]
# ═══════════════════════════════════════════
# LOGGING
# ═══════════════════════════════════════════

# ERROR LOG
log_error = /var/log/mysql/error.log      # Always on. Check this first on issues!

# GENERAL QUERY LOG (logs ALL queries — NEVER in production!)
general_log = OFF                         # Turn ON only for debugging
general_log_file = /var/log/mysql/general.log

# SLOW QUERY LOG — ESSENTIAL for performance tuning
slow_query_log = ON                       # ⭐ ALWAYS enable in production!
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1                       # Log queries slower than 1 second
log_queries_not_using_indexes = ON        # Also log queries without indexes
min_examined_row_limit = 100              # Only log if > 100 rows examined

# BINARY LOG (required for replication & point-in-time recovery)
log_bin = mysql-bin                        # Enable binary logging
binlog_format = ROW                        # ROW-based (safest, default in 8.0)
binlog_expire_logs_seconds = 604800        # Auto-purge after 7 days
max_binlog_size = 256M                     # Max size per binlog file
sync_binlog = 1                            # Sync to disk on every commit (safe)
```

### 6.3 Production-Ready Template (Copy This!)

```ini
# ═══════════════════════════════════════════════════════
# PRODUCTION my.cnf TEMPLATE — 16GB RAM Server
# Adjust values proportionally for your server size
# ═══════════════════════════════════════════════════════

[client]
port = 3306
default-character-set = utf8mb4

[mysql]
prompt = "mysql [\d]> "

[mysqld]
# --- BASIC ---
user = mysql
port = 3306
bind-address = 0.0.0.0
datadir = /var/lib/mysql
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
default_authentication_plugin = caching_sha2_password

# --- INNODB ---
default_storage_engine = InnoDB
innodb_buffer_pool_size = 12G
innodb_buffer_pool_instances = 8
innodb_redo_log_capacity = 4G
innodb_flush_log_at_trx_commit = 1
innodb_log_buffer_size = 64M
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
innodb_file_per_table = ON
innodb_doublewrite = ON
innodb_thread_concurrency = 0

# --- CONNECTIONS ---
max_connections = 500
max_allowed_packet = 64M
wait_timeout = 28800
interactive_timeout = 28800
thread_cache_size = 128

# --- MEMORY ---
tmp_table_size = 256M
max_heap_table_size = 256M
sort_buffer_size = 4M
join_buffer_size = 4M
read_buffer_size = 2M
read_rnd_buffer_size = 4M
table_open_cache = 4000
table_definition_cache = 2000

# --- LOGGING ---
log_error = /var/log/mysql/error.log
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = ON

# --- BINARY LOG ---
log_bin = mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
max_binlog_size = 256M
sync_binlog = 1

# --- SECURITY ---
local_infile = OFF
skip_symbolic_links = ON
sql_mode = STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO

[mysqldump]
quick
max_allowed_packet = 256M
single-transaction
```

---

## 🔥 7. Dynamic vs Static Variables

Some settings can be changed at runtime. Others require a restart.

```sql
-- Check if a variable is dynamic
SELECT VARIABLE_NAME, VARIABLE_SOURCE 
FROM performance_schema.variables_info 
WHERE VARIABLE_NAME = 'innodb_buffer_pool_size';

-- DYNAMIC (change at runtime — no restart needed)
SET GLOBAL max_connections = 1000;
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;
SET GLOBAL innodb_buffer_pool_size = 16 * 1024 * 1024 * 1024;  -- 16GB

-- ⚠️ SET GLOBAL only lasts until restart!
-- To persist across restarts (MySQL 8.0+):
SET PERSIST max_connections = 1000;
-- Saves to /var/lib/mysql/mysqld-auto.cnf (JSON file)

-- SESSION variables (affect only current connection)
SET SESSION sort_buffer_size = 8 * 1024 * 1024;    -- 8MB for this session

-- STATIC (require restart)
-- innodb_redo_log_capacity, innodb_buffer_pool_instances, etc.
-- Must change in my.cnf and restart MySQL
```

```
┌──────────────────────────────────────────────────────────────┐
│  SCOPE OF VARIABLES                                          │
│                                                                │
│  SET SESSION var = value;   ← This connection only            │
│  SET GLOBAL  var = value;   ← All NEW connections (until restart)│
│  SET PERSIST var = value;   ← All connections + survives restart │
│  my.cnf      var = value    ← Applied on every startup          │
│                                                                │
│  Priority: SET PERSIST > my.cnf > compiled defaults            │
└──────────────────────────────────────────────────────────────┘
```

> 💡 **Pro Tip:** `SET PERSIST` in MySQL 8.0 is a game-changer. No more "I changed it with SET GLOBAL but forgot to update my.cnf and lost the change on restart."

---

## 🔥 8. User Management & Security Basics

### Create Users

```sql
-- Create a user (local access only)
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'Str0ng_P@ssw0rd!';

-- Create a user (from any host — use cautiously!)
CREATE USER 'appuser'@'%' IDENTIFIED BY 'Str0ng_P@ssw0rd!';

-- Create a user (from specific IP/subnet)
CREATE USER 'appuser'@'192.168.1.%' IDENTIFIED BY 'Str0ng_P@ssw0rd!';

-- Create with password expiry
CREATE USER 'tempuser'@'localhost' 
    IDENTIFIED BY 'TempPass123!' 
    PASSWORD EXPIRE INTERVAL 90 DAY;
```

### Grant Privileges

```sql
-- Grant ALL on a specific database
GRANT ALL PRIVILEGES ON techmart.* TO 'appuser'@'localhost';

-- Grant specific privileges (RECOMMENDED — principle of least privilege)
GRANT SELECT, INSERT, UPDATE, DELETE ON techmart.* TO 'appuser'@'localhost';

-- Grant read-only access
GRANT SELECT ON techmart.* TO 'readonly_user'@'%';

-- Grant admin privileges (for DBA)
GRANT ALL PRIVILEGES ON *.* TO 'dba_user'@'localhost' WITH GRANT OPTION;

-- Apply changes
FLUSH PRIVILEGES;

-- View grants
SHOW GRANTS FOR 'appuser'@'localhost';
```

### Security Hardening Checklist

```
┌──────────────────────────────────────────────────────────────┐
│  POST-INSTALLATION SECURITY CHECKLIST                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ✅ Run mysql_secure_installation                              │
│  ✅ Set strong root password                                   │
│  ✅ Remove anonymous users                                     │
│  ✅ Disable root remote login                                  │
│  ✅ Remove test database                                       │
│  ✅ Set bind-address = 127.0.0.1 (if no remote access needed) │
│  ✅ Use caching_sha2_password authentication                   │
│  ✅ Enable SSL/TLS for remote connections                      │
│  ✅ Set local_infile = OFF (prevent LOAD DATA LOCAL)           │
│  ✅ Enable slow query log                                      │
│  ✅ Set sql_mode to STRICT_TRANS_TABLES                        │
│  ✅ Use application-specific users (not root!)                 │
│  ✅ Grant minimum required privileges                          │
│  ✅ Set password expiry policies                               │
│  ✅ Enable binary logging (for point-in-time recovery)         │
│  ✅ Regular backups (mysqldump + binlog or Percona XtraBackup) │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔥 9. Backup & Restore — Quick Reference

### mysqldump (Logical Backup — Most Common)

```bash
# Backup single database
mysqldump -u root -p techmart > techmart_backup.sql

# Backup with structure only (no data)
mysqldump -u root -p --no-data techmart > techmart_schema.sql

# Backup with data only (no structure)
mysqldump -u root -p --no-create-info techmart > techmart_data.sql

# Backup ALL databases
mysqldump -u root -p --all-databases > full_backup.sql

# Backup with consistent snapshot (InnoDB — no locks!)
mysqldump -u root -p --single-transaction --routines --triggers techmart > techmart_full.sql

# Compressed backup
mysqldump -u root -p --single-transaction techmart | gzip > techmart_backup.sql.gz
```

### Restore

```bash
# Restore from SQL file
mysql -u root -p techmart < techmart_backup.sql

# Restore compressed backup
gunzip < techmart_backup.sql.gz | mysql -u root -p techmart

# Create database if it doesn't exist, then restore
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS techmart"
mysql -u root -p techmart < techmart_backup.sql
```

> 💡 **For large databases (>10GB):** Use **Percona XtraBackup** (free) or **MySQL Enterprise Backup** (paid) for physical hot backups. They're much faster and don't lock tables.

---

## 🔥 10. MySQL Data Directory — What's on Disk?

```
/var/lib/mysql/                     ← MySQL data directory
├── auto.cnf                        ← Server UUID
├── mysql/                          ← mysql system database
│   ├── user.frm                    ← User table structure
│   └── ...
├── performance_schema/             ← Performance schema tables
├── sys/                            ← sys schema views
├── techmart/                       ← YOUR database!
│   ├── users.ibd                   ← InnoDB: data + index for 'users' table
│   ├── orders.ibd                  ← InnoDB: data + index for 'orders' table
│   └── ...
├── ibdata1                         ← InnoDB system tablespace
├── ib_logfile0                     ← InnoDB redo log (pre-8.0.30)
├── ib_logfile1                     ← InnoDB redo log (pre-8.0.30)
├── #innodb_redo/                   ← InnoDB redo logs (8.0.30+)
├── undo_001                        ← InnoDB undo tablespace
├── undo_002                        ← InnoDB undo tablespace
├── ibtmp1                          ← InnoDB temp tablespace
├── mysql-bin.000001                ← Binary log file
├── mysql-bin.index                 ← Binary log index
├── *.pem                           ← SSL certificate files
└── mysqld-auto.cnf                 ← SET PERSIST changes (MySQL 8.0+)
```

> ⭐ **In MySQL 8.0+:** The `.frm` files (table definitions) are gone! Table metadata is now stored in the InnoDB data dictionary (inside the tablespace files). This is called the **Data Dictionary** — faster, transactional, and crash-safe.

---

## 🧠 Quick Recall — Chapter Summary

| Concept | One-Line Summary |
|---------|-----------------|
| Community Edition | Free, GPL, all core features — use this for learning & most production |
| Enterprise Edition | Paid, adds Thread Pool, Audit, Firewall, Enterprise Backup, Support |
| my.cnf / my.ini | THE config file — `[mysqld]` for server, `[client]` for clients |
| `innodb_buffer_pool_size` | Set to 70-80% of RAM — single most impactful tuning parameter |
| `innodb_flush_log_at_trx_commit` | 1 = safest (ACID), 2 = faster, 0 = fastest (risky) |
| utf8mb4 | Use this, NOT utf8 (which is MySQL's broken 3-byte UTF-8) |
| mysql CLI | The daily-driver tool — learn `\G`, `SOURCE`, `SHOW` commands |
| MySQL Workbench | Visual SQL editor, ER diagram tool, admin GUI — free |
| mysqlsh (MySQL Shell) | Next-gen CLI — supports SQL, JavaScript, Python modes |
| SET PERSIST | MySQL 8.0 feature — saves runtime config changes across restarts |
| Slow Query Log | ALWAYS enable in production — your #1 performance debugging tool |
| mysqldump | Logical backup tool — use `--single-transaction` for InnoDB |
| mysql_secure_installation | First thing to run after installation — security hardening |

---

## ❓ Self-Check Questions

1. What's the difference between MySQL Community and Enterprise Edition?
2. Where is the my.cnf file located on Linux? On Windows?
3. Why should you use `utf8mb4` instead of `utf8` in MySQL?
4. What is `innodb_buffer_pool_size` and how should you size it?
5. What does `innodb_flush_log_at_trx_commit = 1` guarantee?
6. How do you enable the slow query log? Why is it important?
7. What is `SET PERSIST` and how is it different from `SET GLOBAL`?
8. Name 3 things `mysql_secure_installation` does.
9. What's the difference between `[mysqld]`, `[client]`, and `[mysql]` sections in my.cnf?
10. How would you do a consistent backup of an InnoDB database without locking tables?

---

## 🎯 Hands-On Lab

```bash
# 1. Install MySQL (use Docker for quick setup)
docker run --name mysql-lab -e MYSQL_ROOT_PASSWORD=Lab123! -p 3306:3306 -d mysql:8.0

# 2. Connect and explore
docker exec -it mysql-lab mysql -u root -pLab123!

# Inside mysql:
```

```sql
-- 3. Check critical settings
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'sql_mode';

-- 4. Create a database and user
CREATE DATABASE techmart CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'appuser'@'%' IDENTIFIED BY 'App_Pass_123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON techmart.* TO 'appuser'@'%';
FLUSH PRIVILEGES;

-- 5. Test the new user
-- (open a new terminal)
-- docker exec -it mysql-lab mysql -u appuser -pApp_Pass_123! techmart

-- 6. Enable slow query log dynamically
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;

-- 7. Check the data directory
SHOW VARIABLES LIKE 'datadir';

-- 8. Check system databases
SHOW DATABASES;
SELECT TABLE_SCHEMA, COUNT(*) AS table_count
FROM information_schema.TABLES
GROUP BY TABLE_SCHEMA;
```

---

> **Next Chapter** → [2D.3 — MySQL SQL Dialect & Features](./03-MySQL-SQL-Specifics.md)
