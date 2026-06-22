# 🛠️ Chapter 2B.2 — Oracle Installation & Configuration

> **Level:** 🟢 Beginner
> **Time to Master:** ~2-3 hours
> **Prerequisites:** Chapter 2B.1 (Oracle Architecture)

> **"You can't drive a Ferrari if you can't start the engine. Let's get Oracle up and running — and understand every knob and dial."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Know Oracle **editions** and which one to pick
- Install **Oracle XE** (Express Edition) for learning — free!
- Understand the **Oracle Listener** and TNS configuration
- Connect using **SQL\*Plus**, **SQL Developer**, and **JDBC**
- Configure essential **initialization parameters**
- Navigate the Oracle **directory structure** like a pro

---

## 1. Oracle Editions — Know What You're Working With

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    ORACLE DATABASE EDITIONS                           ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐ ║
║  │  🆓 EXPRESS EDITION (XE)                                       │ ║
║  │  • FREE forever                                                 │ ║
║  │  • Max 2 CPU threads, 2 GB RAM, 12 GB user data                │ ║
║  │  • Perfect for: Learning, Development, Small apps               │ ║
║  │  • Latest: Oracle XE 21c                                        │ ║
║  │  ★ THIS IS WHAT YOU SHOULD INSTALL FOR LEARNING                 │ ║
║  └─────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐ ║
║  │  💼 STANDARD EDITION 2 (SE2)                                   │ ║
║  │  • Paid license                                                 │ ║
║  │  • Max 2 sockets, 16 CPU threads                                │ ║
║  │  • No RAC One Node (from 19c+), no partitioning                │ ║
║  │  • For: Small to medium businesses                              │ ║
║  └─────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐ ║
║  │  🏢 ENTERPRISE EDITION (EE)                                    │ ║
║  │  • Full featured — the real beast                               │ ║
║  │  • Unlimited CPU, RAM, data                                     │ ║
║  │  • RAC, Partitioning, Advanced Security, Data Guard             │ ║
║  │  • Additional cost "Options" and "Packs" ($$$$)                │ ║
║  │  • For: Large enterprises, mission-critical systems             │ ║
║  └─────────────────────────────────────────────────────────────────┘ ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐ ║
║  │  ☁️ CLOUD OPTIONS                                               │ ║
║  │  • Oracle Autonomous Database (self-driving!)                   │ ║
║  │  • Oracle Cloud Free Tier — Always Free ATP/ADW                 │ ║
║  │  • For: Cloud-native, serverless, managed database              │ ║
║  └─────────────────────────────────────────────────────────────────┘ ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Quick Decision Table

| Need | Edition | Cost |
|------|---------|------|
| Learning / Practice | **XE 21c** | Free |
| Personal projects | **XE 21c** or **Cloud Free Tier** | Free |
| Small business (< 2 sockets) | **SE2** | ~$17,500/socket |
| Enterprise with HA/Partitioning | **EE** | ~$47,500/processor |
| Cloud managed database | **Autonomous DB** | Pay-as-you-go |

> 💡 **Pro Tip**: Oracle Cloud's **Always Free Tier** gives you an Autonomous Transaction Processing (ATP) instance — full enterprise features, zero cost. Perfect for learning advanced features you can't get in XE.

---

## 2. Installing Oracle XE 21c (Step-by-Step)

### 2.1 On Windows

```
Pre-requisites:
  ✅ Windows 10/11 (64-bit)
  ✅ 2 GB RAM minimum (4 GB recommended)
  ✅ 8 GB free disk space
  ✅ Admin privileges
```

```
Step 1: Download
   → Go to: https://www.oracle.com/database/technologies/xe-downloads.html
   → Download: "Oracle Database 21c Express Edition for Windows x64"
   → File: OracleXE213_Win64.zip (~1.5 GB)

Step 2: Extract & Run Installer
   → Extract the ZIP file
   → Run setup.exe as Administrator
   → Accept license agreement

Step 3: Configuration During Install
   ┌─────────────────────────────────────────────────────────┐
   │  Oracle Home: C:\app\<user>\product\21.0.0\dbhomeXE    │
   │  Oracle Base: C:\app\<user>                              │
   │                                                          │
   │  Set Password for: SYS, SYSTEM, PDBADMIN                │
   │  (Use the SAME password for all — for learning)          │
   │  Example: Oracle123#  (must meet complexity rules)       │
   │                                                          │
   │  Listener Port: 1521 (default)                           │
   │  EM Express Port: 5500                                   │
   └─────────────────────────────────────────────────────────┘

Step 4: Wait for Installation (~10-15 minutes)
   → Creates the database automatically
   → CDB: XE (Container Database)
   → PDB: XEPDB1 (Pluggable Database — where your data goes)

Step 5: Verify
   → Open Command Prompt
   → Type: sqlplus sys/Oracle123#@localhost:1521/XE as sysdba
   → You should see: "Connected to: Oracle Database 21c Express Edition"
```

### 2.2 On Linux (RPM-based: Oracle Linux, CentOS, RHEL)

```bash
# Step 1: Download the RPM from Oracle's website
# File: oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm

# Step 2: Pre-requisite RPM (handles all OS settings)
curl -o oracle-database-preinstall-21c-1.0-1.el8.x86_64.rpm \
  https://yum.oracle.com/repo/OracleLinux/OL8/appstream/x86_64/getPackage/oracle-database-preinstall-21c-1.0-1.el8.x86_64.rpm
sudo rpm -ivh oracle-database-preinstall-21c-*.rpm

# Step 3: Install Oracle XE
sudo rpm -ivh oracle-database-xe-21c-*.rpm

# Step 4: Configure the database (creates CDB + PDB)
sudo /etc/init.d/oracle-xe-21c configure
# Enter password when prompted (for SYS, SYSTEM, PDBADMIN)

# Step 5: Set environment variables
echo 'export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE' >> ~/.bashrc
echo 'export ORACLE_SID=XE' >> ~/.bashrc
echo 'export PATH=$ORACLE_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Step 6: Connect
sqlplus sys/YourPassword@localhost:1521/XE as sysdba
```

### 2.3 Using Docker (Fastest Way! 🚀)

```bash
# Pull Oracle XE image
docker pull container-registry.oracle.com/database/express:21.3.0-xe

# Run Oracle XE container
docker run -d \
  --name oracle-xe \
  -p 1521:1521 \
  -p 5500:5500 \
  -e ORACLE_PWD=Oracle123 \
  -v oracle-data:/opt/oracle/oradata \
  container-registry.oracle.com/database/express:21.3.0-xe

# Wait ~2-3 minutes for database creation, then:
docker exec -it oracle-xe sqlplus sys/Oracle123@localhost:1521/XE as sysdba

# Check logs for startup status:
docker logs -f oracle-xe
```

> 💡 **Pro Tip**: Docker is the fastest way to get Oracle running. No permanent installation, easy cleanup (`docker rm -f oracle-xe`), and you can run multiple versions simultaneously.

---

## 3. Oracle Directory Structure — Where Everything Lives

```
Understanding the directory structure is KEY for troubleshooting.

Windows Default:
┌──────────────────────────────────────────────────────────────────┐
│  C:\app\<username>\                                              │
│  └── product\                                                    │
│      └── 21.0.0\                                                 │
│          └── dbhomeXE\              ← ORACLE_HOME                │
│              ├── bin\               ← Executables (sqlplus, etc) │
│              ├── dbs\               ← Parameter files (spfile)   │
│              ├── network\           ← Net config (listener, tns) │
│              │   └── admin\                                      │
│              │       ├── listener.ora    ← Listener config       │
│              │       ├── tnsnames.ora    ← Connection aliases    │
│              │       └── sqlnet.ora      ← Net settings          │
│              ├── rdbms\             ← RDBMS scripts & admin      │
│              │   └── admin\                                      │
│              ├── sqlplus\           ← SQL*Plus config             │
│              └── assistants\        ← DBCA, NetCA, etc           │
│                                                                   │
│  C:\app\<username>\                                              │
│  └── oradata\                       ← ORACLE_BASE/oradata        │
│      └── XE\                        ← Database files             │
│          ├── system01.dbf           ← SYSTEM tablespace          │
│          ├── sysaux01.dbf           ← SYSAUX tablespace          │
│          ├── undotbs01.dbf          ← UNDO tablespace            │
│          ├── users01.dbf            ← USERS tablespace           │
│          ├── temp01.dbf             ← TEMP tablespace            │
│          ├── redo01.log             ← Redo log group 1           │
│          ├── redo02.log             ← Redo log group 2           │
│          ├── redo03.log             ← Redo log group 3           │
│          └── control01.ctl          ← Control file               │
│                                                                   │
│  C:\app\<username>\diag\            ← Diagnostic files           │
│  └── rdbms\xe\XE\                                                │
│      ├── alert\                     ← ALERT LOG (check daily!)   │
│      │   └── log.xml                                             │
│      └── trace\                     ← Trace files                │
│          └── alert_XE.log           ← Human-readable alert log   │
└──────────────────────────────────────────────────────────────────┘

Linux Default:
┌──────────────────────────────────────────────────────────────────┐
│  ORACLE_BASE = /opt/oracle                                       │
│  ORACLE_HOME = /opt/oracle/product/21c/dbhomeXE                  │
│  ORACLE_SID  = XE                                                │
│                                                                   │
│  /opt/oracle/                                                    │
│  ├── product/21c/dbhomeXE/    ← ORACLE_HOME (same as Windows)   │
│  ├── oradata/XE/              ← Data files, redo, control files  │
│  ├── diag/rdbms/xe/XE/        ← Alert log and traces            │
│  └── fast_recovery_area/      ← Flash Recovery Area (FRA)        │
└──────────────────────────────────────────────────────────────────┘
```

### Key Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `ORACLE_HOME` | Where Oracle software is installed | `/opt/oracle/product/21c/dbhomeXE` |
| `ORACLE_SID` | System Identifier — which instance to connect to | `XE` |
| `ORACLE_BASE` | Base directory for Oracle installations | `/opt/oracle` |
| `PATH` | Must include `$ORACLE_HOME/bin` | Used to find `sqlplus`, `lsnrctl`, etc. |
| `TNS_ADMIN` | Where to find `tnsnames.ora`, `listener.ora` | `$ORACLE_HOME/network/admin` |
| `NLS_LANG` | Character set and language settings | `AMERICAN_AMERICA.AL32UTF8` |
| `LD_LIBRARY_PATH` | Shared libraries (Linux only) | `$ORACLE_HOME/lib` |

```bash
# Set all at once (add to .bashrc or .bash_profile)
export ORACLE_BASE=/opt/oracle
export ORACLE_HOME=$ORACLE_BASE/product/21c/dbhomeXE
export ORACLE_SID=XE
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export TNS_ADMIN=$ORACLE_HOME/network/admin
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
```

---

## 4. Oracle Listener — The Gatekeeper

The Listener is a **separate process** that listens for incoming client connections and routes them to the right database instance.

```
Without a Listener, NO remote connections are possible!

  ┌──────────────┐
  │ Client App   │
  │ (SQL Dev,    │    "I want to connect to XE on port 1521"
  │  JDBC, etc.) │──────────────────────────────┐
  └──────────────┘                              │
                                                 ▼
                                   ┌─────────────────────┐
                                   │    LISTENER          │
                                   │    (lsnrctl)         │
                                   │                      │
                                   │  Port: 1521          │
                                   │  Protocol: TCP       │
                                   │                      │
                                   │  Registered DBs:     │
                                   │  • XE (Instance)     │
                                   │  • XEPDB1 (PDB)      │
                                   └──────────┬──────────┘
                                              │
                                   Routes to correct instance
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │  Oracle Instance     │
                                   │  (SGA + Processes)   │
                                   │                      │
                                   │  Spawns a dedicated  │
                                   │  server process for  │
                                   │  this client         │
                                   └─────────────────────┘
```

### listener.ora — Listener Configuration

```
# Location: $ORACLE_HOME/network/admin/listener.ora

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

# Optional: Static registration (if dynamic registration fails)
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = XE)
      (ORACLE_HOME = /opt/oracle/product/21c/dbhomeXE)
      (SID_NAME = XE)
    )
  )
```

### Listener Management Commands

```bash
# Check listener status
lsnrctl status

# Start listener
lsnrctl start

# Stop listener
lsnrctl stop

# Reload configuration without restart
lsnrctl reload

# Show registered services
lsnrctl services
```

### Expected Output of `lsnrctl status`

```
LSNRCTL> status
Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 21.0.0.0.0
Start Date                02-JUN-2026 09:00:00
Uptime                    5 days 3 hr. 22 min. 15 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /opt/oracle/product/21c/dbhomeXE/network/admin/listener.ora
Listener Log File         /opt/oracle/diag/tnslsnr/hostname/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521)))
Services Summary...
Service "XE" has 1 instance(s).
  Instance "XE", status READY, has 1 handler(s) for this service...
Service "XEPDB1" has 1 instance(s).
  Instance "XE", status READY, has 1 handler(s) for this service...
The command completed successfully
```

> 💡 **Troubleshooting**: If `lsnrctl status` shows services but your app can't connect, check **firewall rules** (port 1521) and **HOST** setting (use actual hostname, not `localhost`, for remote connections).

---

## 5. TNS Names — Connection Aliases

`tnsnames.ora` maps friendly names to full database connection details. It's like a **phonebook** for databases.

### tnsnames.ora Configuration

```
# Location: $ORACLE_HOME/network/admin/tnsnames.ora

# Connect to the Container Database (CDB)
XE =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = XE)
    )
  )

# Connect to the Pluggable Database (PDB) — YOUR DATA GOES HERE
XEPDB1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = XEPDB1)
    )
  )

# Remote database example
PROD_DB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = prod-server.company.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = PRODDB)
    )
  )
```

### Connection Formats — TNS vs Easy Connect

```sql
-- Method 1: TNS Name (requires tnsnames.ora entry)
sqlplus scott/tiger@XE
sqlplus sys/password@XEPDB1 as sysdba

-- Method 2: Easy Connect (no tnsnames.ora needed!)
sqlplus scott/tiger@localhost:1521/XEPDB1
sqlplus sys/password@//prod-server:1521/PRODDB as sysdba

-- Method 3: Local connection (ORACLE_SID must be set)
export ORACLE_SID=XE
sqlplus / as sysdba       -- OS authentication, no password needed
sqlplus sys/password as sysdba

-- Test TNS resolution
tnsping XE
tnsping XEPDB1
```

### Easy Connect Syntax Breakdown

```
username/password@host:port/service_name

Examples:
  sqlplus hr/hr@localhost:1521/XEPDB1
  sqlplus sys/Oracle123@192.168.1.100:1521/XE as sysdba
  sqlplus admin/pass@db.example.com:1521/SALES_PDB

No tnsnames.ora needed! Great for quick connections.
```

---

## 6. SQL\*Plus — The Original Oracle CLI

SQL\*Plus has been around since Oracle's early days. It's not pretty, but it's **always available** and every DBA must know it.

### Essential SQL\*Plus Commands

```sql
-- Connect to database
sqlplus sys/password@XE as sysdba
sqlplus hr/hr@XEPDB1

-- Once connected:

-- ═══════════════════════════════════════════
-- DISPLAY & FORMATTING
-- ═══════════════════════════════════════════

-- Set column width for readable output
COLUMN employee_name FORMAT A30
COLUMN salary FORMAT 999,999.99
COLUMN department FORMAT A20

-- Set line width and page size
SET LINESIZE 200
SET PAGESIZE 50

-- Show/hide column headers
SET HEADING ON

-- Timing of queries
SET TIMING ON

-- Auto-print DBMS_OUTPUT
SET SERVEROUTPUT ON

-- ═══════════════════════════════════════════
-- NAVIGATION & INFO
-- ═══════════════════════════════════════════

-- Show current user
SHOW USER

-- Show current container (CDB or PDB)
SHOW CON_NAME

-- Show parameter value
SHOW PARAMETER db_name
SHOW PARAMETER sga_target
SHOW PARAMETER pga_aggregate_target

-- Describe a table
DESC employees
DESC dba_tables

-- ═══════════════════════════════════════════
-- SCRIPTING
-- ═══════════════════════════════════════════

-- Run a SQL script
@/path/to/script.sql
@@relative_script.sql

-- Save output to file (spool)
SPOOL /tmp/output.txt
SELECT * FROM employees;
SPOOL OFF

-- Edit last SQL statement in editor
EDIT

-- Re-run last SQL statement
/

-- ═══════════════════════════════════════════
-- SESSION MANAGEMENT
-- ═══════════════════════════════════════════

-- Switch to a PDB
ALTER SESSION SET CONTAINER = XEPDB1;

-- Change NLS settings for session
ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI:SS';

-- Exit
EXIT
QUIT
```

### SQL\*Plus Login Script (login.sql)

Create this file in `$ORACLE_HOME/sqlplus/admin/` or your working directory:

```sql
-- login.sql — Runs automatically when SQL*Plus starts

SET PAGESIZE 50
SET LINESIZE 200
SET TIMING ON
SET SERVEROUTPUT ON SIZE UNLIMITED
SET SQLPROMPT "_user'@'_connect_identifier > "

ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI:SS';

-- Now your prompt shows: SYS@XE >
```

> 💡 **Pro Tip**: The `login.sql` file saves you from typing the same SET commands every time you start SQL\*Plus. Every experienced DBA has one.

---

## 7. Oracle SQL Developer — The Modern GUI

SQL Developer is Oracle's **free** graphical tool. It's Java-based, cross-platform, and much friendlier than SQL\*Plus.

### Setting Up a Connection

```
┌──────────────────────────────────────────────────────────────┐
│          NEW DATABASE CONNECTION                              │
│                                                               │
│  Connection Name:  [ XE_Local              ]                 │
│                                                               │
│  ─── Authentication ───                                       │
│  Username:         [ sys                   ]                 │
│  Password:         [ ••••••••              ]                 │
│  Role:             [ SYSDBA ▼              ]                 │
│  ☑ Save Password                                             │
│                                                               │
│  ─── Connection Type: Basic ───                               │
│  Hostname:         [ localhost              ]                │
│  Port:             [ 1521                   ]                │
│  Service Name: ●   [ XE                    ]                 │
│  SID:          ○   [                       ]                 │
│                                                               │
│  ─── OR Connection Type: TNS ───                              │
│  Network Alias:    [ XEPDB1 ▼              ]                 │
│                                                               │
│  [ Test ]  [ Connect ]  [ Cancel ]                           │
└──────────────────────────────────────────────────────────────┘
```

### Key SQL Developer Features

| Feature | Where to Find | What It Does |
|---------|--------------|--------------|
| **SQL Worksheet** | Tools → SQL Worksheet | Write & execute SQL/PL/SQL |
| **Explain Plan** | F10 or Explain Plan button | Visual execution plan |
| **DBA Panel** | View → DBA | Manage storage, sessions, security |
| **Data Modeler** | File → Data Modeler | Design ER diagrams |
| **Export Wizard** | Tools → Database Export | Export DDL, data, or both |
| **Import Data** | Right-click table → Import Data | Load CSV, Excel into tables |
| **Reports** | View → Reports | Pre-built reports (all tables, all indexes, etc.) |
| **Snippets** | View → Snippets | Code templates |

---

## 8. JDBC Connection — Connecting from Applications

### JDBC URL Formats

```java
// Format 1: Thin driver with Service Name (RECOMMENDED)
String url = "jdbc:oracle:thin:@//hostname:1521/service_name";
// Example:
String url = "jdbc:oracle:thin:@//localhost:1521/XEPDB1";

// Format 2: Thin driver with SID (legacy, avoid for new projects)
String url = "jdbc:oracle:thin:@hostname:1521:SID";
// Example:
String url = "jdbc:oracle:thin:@localhost:1521:XE";

// Format 3: Using TNS entry
String url = "jdbc:oracle:thin:@XE";  // Requires tnsnames.ora in TNS_ADMIN

// Format 4: Full TNS descriptor (when tnsnames.ora isn't available)
String url = "jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)" +
             "(HOST=localhost)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=XEPDB1)))";
```

### Java Connection Example

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

public class OracleConnect {
    public static void main(String[] args) throws Exception {
        // JDBC URL
        String url = "jdbc:oracle:thin:@//localhost:1521/XEPDB1";
        String user = "hr";
        String password = "hr";
        
        // Connect
        try (Connection conn = DriverManager.getConnection(url, user, password)) {
            System.out.println("Connected to Oracle!");
            System.out.println("Database version: " + 
                conn.getMetaData().getDatabaseProductVersion());
            
            // Execute a query
            try (Statement stmt = conn.createStatement();
                 ResultSet rs = stmt.executeQuery(
                     "SELECT employee_id, first_name, salary FROM employees " +
                     "WHERE ROWNUM <= 5")) {
                while (rs.next()) {
                    System.out.printf("%-6d %-15s %,.2f%n",
                        rs.getInt("employee_id"),
                        rs.getString("first_name"),
                        rs.getDouble("salary"));
                }
            }
        }
    }
}
```

### Python Connection (cx_Oracle / python-oracledb)

```python
# Modern approach: python-oracledb (recommended, replaces cx_Oracle)
# pip install oracledb

import oracledb

# Thin mode (no Oracle Client needed!)
conn = oracledb.connect(
    user="hr",
    password="hr",
    dsn="localhost:1521/XEPDB1"
)

cursor = conn.cursor()
cursor.execute("SELECT employee_id, first_name, salary FROM employees WHERE ROWNUM <= 5")

for emp_id, name, salary in cursor:
    print(f"{emp_id:6d} {name:15s} {salary:>10,.2f}")

cursor.close()
conn.close()
```

### Connection String Cheat Sheet

| Language | Connection Format |
|----------|------------------|
| **SQL\*Plus** | `sqlplus hr/hr@localhost:1521/XEPDB1` |
| **Java JDBC** | `jdbc:oracle:thin:@//localhost:1521/XEPDB1` |
| **Python** | `oracledb.connect(dsn="localhost:1521/XEPDB1")` |
| **Node.js** | `oracledb.getConnection({connectString: "localhost:1521/XEPDB1"})` |
| **C# (.NET)** | `Data Source=localhost:1521/XEPDB1;User Id=hr;Password=hr;` |
| **Go** | `godror.ConnectionParams{DSN: "localhost:1521/XEPDB1"}` |

---

## 9. Essential Initialization Parameters

Oracle has **hundreds** of parameters. Here are the ones that matter most:

### Memory Parameters

```sql
-- ═══════════════════════════════════════════════════
-- MEMORY MANAGEMENT (Most Important!)
-- ═══════════════════════════════════════════════════

-- Option 1: Automatic Memory Management (AMM) — Let Oracle decide
-- Oracle manages both SGA and PGA automatically
ALTER SYSTEM SET MEMORY_TARGET = 4G SCOPE=SPFILE;
ALTER SYSTEM SET MEMORY_MAX_TARGET = 6G SCOPE=SPFILE;

-- Option 2: Automatic Shared Memory Management (ASMM) — More control
-- You set SGA and PGA targets, Oracle manages sub-components
ALTER SYSTEM SET SGA_TARGET = 3G SCOPE=SPFILE;
ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 1G SCOPE=SPFILE;

-- Option 3: Manual — Full control (DBA experts only)
ALTER SYSTEM SET DB_CACHE_SIZE = 2G SCOPE=SPFILE;
ALTER SYSTEM SET SHARED_POOL_SIZE = 500M SCOPE=SPFILE;
ALTER SYSTEM SET LOG_BUFFER = 50M SCOPE=SPFILE;
ALTER SYSTEM SET LARGE_POOL_SIZE = 100M SCOPE=SPFILE;
```

```
Which memory management mode to use?

  ┌────────────────────────────────────────────────────────┐
  │  Learning / Development?  → AMM (MEMORY_TARGET)       │
  │  Production (Linux)?      → ASMM (SGA + PGA targets)  │
  │  Expert DBA tuning?       → Manual (individual sizes)  │
  └────────────────────────────────────────────────────────┘

  Note: On Linux with HugePages, AMM doesn't work.
        Use ASMM instead. This is the #1 gotcha in production.
```

### Critical Parameters Table

| Parameter | Default | Description | Recommendation |
|-----------|---------|-------------|----------------|
| `DB_BLOCK_SIZE` | 8192 (8KB) | Block size — set at DB creation, CANNOT change later | 8KB for OLTP, 16KB/32KB for DW |
| `PROCESSES` | 100-300 | Max concurrent OS processes | 300-500 for production |
| `SESSIONS` | 1.5 × PROCESSES + 22 | Max concurrent sessions | Auto-calculated |
| `OPEN_CURSORS` | 50-300 | Max open cursors per session | 300-500 |
| `DB_FILES` | 200 | Max number of data files | 500-1000 for large DBs |
| `UNDO_RETENTION` | 900 (seconds) | How long to keep undo data | 1800-3600 for Flashback |
| `OPTIMIZER_MODE` | ALL_ROWS | Query optimizer strategy | ALL_ROWS (default) is fine |
| `CONTROL_FILES` | Varies | Control file locations | 3 copies on different disks! |
| `LOG_ARCHIVE_DEST_1` | None | Where to archive redo logs | Set for ARCHIVELOG mode |
| `DB_RECOVERY_FILE_DEST` | Varies | Flash Recovery Area location | Set to fast storage |

### Viewing & Modifying Parameters

```sql
-- View all parameters
SHOW PARAMETER;

-- Search for specific parameter
SHOW PARAMETER memory;
SHOW PARAMETER sga;
SHOW PARAMETER optimizer;

-- View parameter with full details
SELECT name, value, isdefault, ismodified, issys_modifiable
FROM v$parameter
WHERE name LIKE '%memory%';

-- Modify parameter
-- SCOPE options:
--   MEMORY  → Change now, lost after restart
--   SPFILE  → Change in parameter file, takes effect after restart
--   BOTH    → Change now AND save to spfile (best of both)

ALTER SYSTEM SET open_cursors = 500 SCOPE=BOTH;
ALTER SYSTEM SET pga_aggregate_target = 2G SCOPE=BOTH;

-- Some parameters require restart (static parameters)
ALTER SYSTEM SET processes = 500 SCOPE=SPFILE;  -- Requires restart
-- Then: SHUTDOWN IMMEDIATE; STARTUP;
```

### Parameter File: SPFILE vs PFILE

```
SPFILE (Server Parameter File) — Binary, Server-managed ⭐
  • Location: $ORACLE_HOME/dbs/spfileXE.ora (Linux)
  •           %ORACLE_HOME%\database\SPFILEXE.ORA (Windows)
  • Modified with: ALTER SYSTEM SET ... SCOPE=SPFILE;
  • Cannot be edited with a text editor (binary!)
  • Preferred for production

PFILE (Parameter File) — Text, Human-readable
  • Location: $ORACLE_HOME/dbs/initXE.ora
  • Edited with: Any text editor
  • Used for: troubleshooting, when spfile is corrupted

-- Create pfile from spfile (backup!)
CREATE PFILE='/tmp/init_backup.ora' FROM SPFILE;

-- Create spfile from pfile
CREATE SPFILE FROM PFILE='/tmp/init_backup.ora';
```

---

## 10. Creating Your First User & Schema

In Oracle 21c with Multitenant, you work inside the **Pluggable Database (PDB)**:

```sql
-- Connect as SYSDBA
sqlplus sys/Oracle123@localhost:1521/XEPDB1 as sysdba

-- Create a tablespace for your user's data
CREATE TABLESPACE learn_data
  DATAFILE '/opt/oracle/oradata/XE/XEPDB1/learn_data01.dbf'
  SIZE 100M
  AUTOEXTEND ON
  NEXT 50M
  MAXSIZE 2G;

-- Create a user (in 21c PDB, no need for C## prefix)
CREATE USER student IDENTIFIED BY Student123#
  DEFAULT TABLESPACE learn_data
  TEMPORARY TABLESPACE temp
  QUOTA UNLIMITED ON learn_data;

-- Grant necessary privileges
GRANT CREATE SESSION TO student;          -- Can connect
GRANT CREATE TABLE TO student;            -- Can create tables
GRANT CREATE VIEW TO student;             -- Can create views
GRANT CREATE SEQUENCE TO student;         -- Can create sequences
GRANT CREATE PROCEDURE TO student;        -- Can create procedures
GRANT CREATE TRIGGER TO student;          -- Can create triggers
GRANT CREATE SYNONYM TO student;          -- Can create synonyms

-- OR use a role (easier):
GRANT CONNECT, RESOURCE TO student;

-- Test the new user
CONNECT student/Student123#@localhost:1521/XEPDB1

-- Verify
SELECT USER FROM dual;
-- Output: STUDENT
```

### Unlock the HR Sample Schema (Great for Practice!)

```sql
-- Connect as SYSDBA to the PDB
sqlplus sys/Oracle123@localhost:1521/XEPDB1 as sysdba

-- Unlock HR user (it's locked by default)
ALTER USER hr IDENTIFIED BY hr ACCOUNT UNLOCK;

-- Grant connect privilege
GRANT CREATE SESSION TO hr;

-- Test
CONNECT hr/hr@localhost:1521/XEPDB1

-- Check available tables
SELECT table_name FROM user_tables ORDER BY table_name;
-- COUNTRIES, DEPARTMENTS, EMPLOYEES, JOBS, JOB_HISTORY,
-- LOCATIONS, REGIONS

-- Query employees
SELECT first_name, last_name, salary, department_id
FROM employees
WHERE ROWNUM <= 5;
```

---

## 11. Enterprise Manager Express (EM Express) — Web-Based Monitoring

Oracle XE comes with a lightweight web-based monitoring tool:

```
URL: https://localhost:5500/em

Login:
  Username: sys
  Password: <your_password>
  Container: CDB or PDB name
  As: SYSDBA

Features:
  ✅ Performance Hub (real-time monitoring)
  ✅ SQL Monitor (running SQL details)
  ✅ Storage overview
  ✅ Session management
  ✅ Initialization parameters
  ❌ No job scheduling (use DBMS_SCHEDULER)
  ❌ No backup management (use RMAN CLI)
```

```sql
-- Check if EM Express is configured
SELECT dbms_xdb_config.gethttpsport() FROM dual;
-- Should return 5500

-- If not configured:
EXEC DBMS_XDB_CONFIG.SETHTTPSPORT(5500);
```

---

## 12. Common Post-Installation Tasks Checklist

```
┌──────────────────────────────────────────────────────────────┐
│          POST-INSTALLATION CHECKLIST                          │
│                                                               │
│  ✅  1. Verify instance is running                           │
│         SELECT status FROM v$instance;                       │
│                                                               │
│  ✅  2. Enable ARCHIVELOG mode (for production recovery)     │
│         SHUTDOWN IMMEDIATE;                                   │
│         STARTUP MOUNT;                                        │
│         ALTER DATABASE ARCHIVELOG;                            │
│         ALTER DATABASE OPEN;                                  │
│                                                               │
│  ✅  3. Multiplex control files (at least 3 copies)          │
│         ALTER SYSTEM SET control_files =                      │
│           '/disk1/control01.ctl',                            │
│           '/disk2/control02.ctl',                            │
│           '/disk3/control03.ctl' SCOPE=SPFILE;               │
│                                                               │
│  ✅  4. Multiplex redo log groups (at least 2 members each)  │
│         ALTER DATABASE ADD LOGFILE MEMBER                     │
│           '/disk2/redo01b.log' TO GROUP 1;                   │
│                                                               │
│  ✅  5. Set up automatic backups (RMAN)                       │
│         See Chapter 2B.7 — Oracle Administration              │
│                                                               │
│  ✅  6. Configure memory (SGA_TARGET + PGA_AGGREGATE_TARGET) │
│                                                               │
│  ✅  7. Create application users (NOT sys/system!)            │
│                                                               │
│  ✅  8. Lock unused default accounts                          │
│         ALTER USER dbsnmp ACCOUNT LOCK;                      │
│         ALTER USER appqossys ACCOUNT LOCK;                   │
│                                                               │
│  ✅  9. Set alert log monitoring                              │
│         -- Know where your alert log is:                      │
│         SHOW PARAMETER diagnostic_dest;                       │
│                                                               │
│  ✅ 10. Test connectivity from application servers            │
│         tnsping XEPDB1                                       │
│         sqlplus hr/hr@remote_host:1521/XEPDB1                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🧠 Chapter Summary — Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│        ORACLE INSTALLATION — QUICK REFERENCE                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Edition for Learning: Oracle XE 21c (FREE)                  │
│  Alternative: Oracle Cloud Always Free Tier                   │
│  Fastest Setup: Docker container                              │
│                                                               │
│  Key Environment Variables:                                   │
│    ORACLE_HOME  → Software location                          │
│    ORACLE_SID   → Instance name (XE)                         │
│    ORACLE_BASE  → Base directory                              │
│    PATH         → Include $ORACLE_HOME/bin                   │
│                                                               │
│  Connection Methods:                                          │
│    SQL*Plus: sqlplus user/pass@host:1521/service              │
│    Easy Connect: user/pass@host:port/service_name            │
│    TNS: user/pass@tns_alias                                  │
│    JDBC: jdbc:oracle:thin:@//host:1521/service               │
│                                                               │
│  Listener:                                                    │
│    Config: $ORACLE_HOME/network/admin/listener.ora           │
│    Commands: lsnrctl start/stop/status/reload                │
│    Default Port: 1521                                        │
│                                                               │
│  Parameter File:                                              │
│    SPFILE (binary, preferred) vs PFILE (text, backup)        │
│    Modify: ALTER SYSTEM SET param=val SCOPE=BOTH;            │
│                                                               │
│  Memory Modes:                                                │
│    AMM: MEMORY_TARGET (easiest)                              │
│    ASMM: SGA_TARGET + PGA_AGGREGATE_TARGET (production)      │
│    Manual: Individual pool sizes (expert)                     │
│                                                               │
│  First Steps After Install:                                   │
│    1. Connect as sysdba                                      │
│    2. Enable ARCHIVELOG                                      │
│    3. Multiplex control files & redo logs                    │
│    4. Create application user (not sys!)                     │
│    5. Unlock HR schema for practice                          │
└──────────────────────────────────────────────────────────────┘
```

---

## ❓ Common Installation Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `ORA-12541: TNS:no listener` | Listener not running | `lsnrctl start` |
| `ORA-12514: TNS:listener does not know of service` | Service not registered | Wait 60s (dynamic registration) or add static registration |
| `ORA-01034: ORACLE not available` | Instance not started | `sqlplus / as sysdba` then `STARTUP` |
| `ORA-01031: insufficient privileges` | Not in dba/oinstall group (Linux) | Add user to `dba` group: `usermod -aG dba oracle` |
| `ORA-12547: TNS:lost contact` | Wrong ORACLE_HOME or missing libs | Verify env variables, run `oracle` binary manually |
| `ORA-00845: MEMORY_TARGET not supported` | Linux HugePages + AMM conflict | Use ASMM instead (SGA_TARGET + PGA_AGGREGATE_TARGET) |
| Can't connect remotely | Firewall blocking port 1521 | Open port: `firewall-cmd --add-port=1521/tcp --permanent` |
| `ORA-65096: common user name invalid` | Creating user in CDB without C## prefix | Connect to PDB first: `ALTER SESSION SET CONTAINER = XEPDB1;` |

---

> **Next Chapter**: [2B.3 — Oracle SQL Dialect Specifics →](./03-Oracle-SQL-Specifics.md)
