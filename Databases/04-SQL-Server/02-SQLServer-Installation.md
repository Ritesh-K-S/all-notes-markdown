# 🛠️ Chapter 2C.2 — SQL Server Installation & Tools (SSMS, Azure Data Studio)

> **"A craftsman is only as good as their tools. Before you write a single query, set up your workbench like a pro."**

> **Level:** 🟢 Beginner Friendly
> **Time to Master:** ~2-3 hours (including installation)
> **Prerequisites:** Chapter 2C.1 (SQL Server Architecture — optional but recommended)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Know every **SQL Server edition** and pick the right one
- Install SQL Server on **Windows** (and know the Linux/Docker options)
- Master **SSMS** — the most powerful SQL Server tool
- Set up **Azure Data Studio** — the modern, cross-platform alternative
- Use **sqlcmd** — the command-line warrior
- Configure **SQL Server Configuration Manager** like a DBA
- Connect to SQL Server from **any client** without hesitation

---

## 🏷️ 1. SQL Server Editions — Which One Do You Need?

SQL Server comes in **multiple editions**. Picking the wrong one = wasted money or missing features.

```
╔════════════════════════════════════════════════════════════════════════╗
║                    SQL SERVER EDITIONS (2022)                          ║
║                                                                        ║
║   ┌─────────────────────────────────────────────────────────────┐     ║
║   │  🟢 DEVELOPER EDITION                                       │     ║
║   │  • FREE — Full Enterprise features!                         │     ║
║   │  • For development & testing ONLY (not production)          │     ║
║   │  • 🏆 Best choice for learning!                             │     ║
║   └─────────────────────────────────────────────────────────────┘     ║
║                                                                        ║
║   ┌─────────────────────────────────────────────────────────────┐     ║
║   │  🟢 EXPRESS EDITION                                          │     ║
║   │  • FREE — for small production apps                          │     ║
║   │  • Limits: 10 GB DB size, 1 GB RAM, 4 cores                │     ║
║   │  • Perfect for small websites, prototypes, learning         │     ║
║   └─────────────────────────────────────────────────────────────┘     ║
║                                                                        ║
║   ┌─────────────────────────────────────────────────────────────┐     ║
║   │  🟡 STANDARD EDITION                                         │     ║
║   │  • Paid — for medium workloads                               │     ║
║   │  • Limits: 128 GB RAM, 24 cores                             │     ║
║   │  • Basic HA (log shipping, basic AG)                        │     ║
║   │  • Most SMBs use this                                        │     ║
║   └─────────────────────────────────────────────────────────────┘     ║
║                                                                        ║
║   ┌─────────────────────────────────────────────────────────────┐     ║
║   │  🔴 ENTERPRISE EDITION                                       │     ║
║   │  • 💰 Expensive — for mission-critical workloads            │     ║
║   │  • No limits on RAM/CPU                                      │     ║
║   │  • All features: Always On AG, In-Memory OLTP,              │     ║
║   │    Compression, Partitioning, Columnstore, etc.              │     ║
║   │  • Banks, hospitals, Fortune 500                             │     ║
║   └─────────────────────────────────────────────────────────────┘     ║
║                                                                        ║
║   ┌─────────────────────────────────────────────────────────────┐     ║
║   │  ☁️ AZURE SQL                                                │     ║
║   │  • Cloud-managed (PaaS) — no installation needed            │     ║
║   │  • Pay-per-use, auto-scaling, built-in HA                   │     ║
║   │  • Options: SQL Database, Managed Instance, VM              │     ║
║   └─────────────────────────────────────────────────────────────┘     ║
╚════════════════════════════════════════════════════════════════════════╝
```

### Quick Comparison Table

| Feature | Express (Free) | Developer (Free) | Standard | Enterprise |
|---------|---------------|-------------------|----------|------------|
| **Price** | $0 | $0 | ~$3,945/2-core | ~$15,123/2-core |
| **Max DB Size** | 10 GB | Unlimited | 524 PB | 524 PB |
| **Max RAM** | 1 GB | OS Max | 128 GB | OS Max |
| **Max CPU** | 4 cores | OS Max | 24 cores | OS Max |
| **Always On AG** | ❌ | ✅ (dev only) | Basic only | ✅ Full |
| **In-Memory OLTP** | ❌ | ✅ (dev only) | Limited | ✅ Full |
| **Columnstore** | ❌ | ✅ (dev only) | ✅ | ✅ |
| **Compression** | ❌ | ✅ (dev only) | ❌ Row only | ✅ Full |
| **Production Use** | ✅ (limited) | ❌ | ✅ | ✅ |

> 💡 **Pro Tip for Learners:** Install **Developer Edition** — it's 100% free and has EVERY Enterprise feature. You get the full power to learn without spending a penny.

---

## 🖥️ 2. Installing SQL Server — Step by Step

### Option A: Windows Installation (Recommended for Beginners)

#### Step 1: Download

```
Go to: https://www.microsoft.com/en-us/sql-server/sql-server-downloads

Choose: "Developer" (free, full-featured)
   OR:  "Express" (free, limited)

Download the installer (~~6 MB bootstrapper)
```

#### Step 2: Installation Type

```
╔════════════════════════════════════════════════════════════╗
║           INSTALLATION TYPE SELECTION                      ║
║                                                            ║
║   ┌────────────────────────────────────────────────────┐  ║
║   │  🟢 BASIC                                          │  ║
║   │  • Quick install with default settings              │  ║
║   │  • Best for: Learning, first-time setup            │  ║
║   │  • Installs: Database Engine only                   │  ║
║   └────────────────────────────────────────────────────┘  ║
║                                                            ║
║   ┌────────────────────────────────────────────────────┐  ║
║   │  🟡 CUSTOM                                         │  ║
║   │  • Choose exactly what to install                   │  ║
║   │  • Best for: Production, specific needs            │  ║
║   │  • Choose: Engine, SSIS, SSRS, SSAS, etc.          │  ║
║   └────────────────────────────────────────────────────┘  ║
║                                                            ║
║   ┌────────────────────────────────────────────────────┐  ║
║   │  📥 DOWNLOAD MEDIA                                  │  ║
║   │  • Downloads the full ISO for offline install       │  ║
║   │  • Best for: Servers without internet access        │  ║
║   └────────────────────────────────────────────────────┘  ║
║                                                            ║
║   👉 For learning: Choose BASIC                           ║
║   👉 For production: Choose CUSTOM                        ║
╚════════════════════════════════════════════════════════════╝
```

#### Step 3: Key Installation Decisions (Custom Install)

```
┌────────────────────────────────────────────────────────────────┐
│              KEY DECISIONS DURING INSTALL                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1️⃣ INSTANCE NAME                                              │
│     • Default Instance: MSSQLSERVER (connect via "ServerName") │
│     • Named Instance: e.g., DEV2026 (connect via                │
│       "ServerName\DEV2026")                                     │
│     👉 For learning: Use Default Instance                       │
│                                                                 │
│  2️⃣ AUTHENTICATION MODE                                        │
│     • Windows Authentication: Uses your Windows login          │
│     • Mixed Mode: Windows + SQL Server logins (sa account)     │
│     👉 For learning: Choose Mixed Mode (more flexible)          │
│     👉 Set a STRONG sa password! ⚠️                            │
│                                                                 │
│  3️⃣ SERVICE ACCOUNTS                                           │
│     • Default: NT Service\MSSQLSERVER (fine for dev)           │
│     • Production: Use a dedicated domain service account       │
│                                                                 │
│  4️⃣ COLLATION (Sorting/Comparison rules)                       │
│     • Default: SQL_Latin1_General_CP1_CI_AS                    │
│     • CI = Case Insensitive, AS = Accent Sensitive             │
│     ⚠️ CANNOT be easily changed after install!                 │
│     👉 Pick wisely — this affects all string comparisons       │
│                                                                 │
│  5️⃣ DATA DIRECTORIES                                           │
│     • Separate data (.mdf), log (.ldf), and tempdb files       │
│       on different drives for production                        │
│     • Dev: defaults are fine                                    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Option B: Docker Installation (Linux/Mac/Cross-Platform)

```bash
# Pull the SQL Server 2022 Docker image
docker pull mcr.microsoft.com/mssql/server:2022-latest

# Run SQL Server in a container
docker run -e "ACCEPT_EULA=Y" \
           -e "MSSQL_SA_PASSWORD=YourStr0ng!Password" \
           -p 1433:1433 \
           --name sql2022 \
           --hostname sql2022 \
           -d mcr.microsoft.com/mssql/server:2022-latest

# Verify it's running
docker ps

# Connect using sqlcmd inside the container
docker exec -it sql2022 /opt/mssql-tools18/bin/sqlcmd \
    -S localhost -U sa -P "YourStr0ng!Password" -C

# You're in! Try:
SELECT @@VERSION;
GO
```

> 💡 **Pro Tip:** Docker is the FASTEST way to get SQL Server running — under 2 minutes. No installation wizard, no reboots, no Windows required.

### Option C: Azure SQL (Cloud — Zero Installation)

```
1. Go to https://portal.azure.com
2. Create Resource → SQL Database
3. Choose: Serverless tier (auto-pause = saves money 💰)
4. Configure firewall rules
5. Connect from SSMS or Azure Data Studio
6. Done! No patches, no OS, no maintenance. ☁️
```

---

## 🔧 3. SQL Server Management Studio (SSMS) — The Power Tool

**SSMS** is the **#1 tool** for SQL Server. Every DBA and SQL developer uses it daily.

```
╔══════════════════════════════════════════════════════════════════════╗
║                           SSMS INTERFACE                             ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │  Menu Bar: File │ Edit │ View │ Query │ Tools │ Help          │   ║
║  ├──────────────────────────────────────────────────────────────┤   ║
║  │  Toolbar: ▶ Execute │ ✓ Parse │ 📊 Plan │ 🔌 Connect       │   ║
║  ├──────────┬───────────────────────────────────────────────────┤   ║
║  │          │                                                    │   ║
║  │ Object   │   Query Editor                                    │   ║
║  │ Explorer │                                                    │   ║
║  │          │   SELECT TOP 100 *                                │   ║
║  │ ▼ Server │   FROM Sales.Orders                               │   ║
║  │   ▼ DBs  │   WHERE OrderDate > '2026-01-01'                 │   ║
║  │     ▼ Adv│   ORDER BY TotalAmount DESC;                      │   ║
║  │       Tab│                                                    │   ║
║  │       Vie│                                                    │   ║
║  │       SP │                                                    │   ║
║  │       Fun│                                                    │   ║
║  │   ▼ Secu│                                                    │   ║
║  │   ▼ Agent│───────────────────────────────────────────────────┤   ║
║  │          │   Results Grid / Messages / Execution Plan        │   ║
║  │          │                                                    │   ║
║  │          │   OrderID │ Customer │ Total  │ Date              │   ║
║  │          │   10248   │ VINET    │ 440.00 │ 2026-01-04       │   ║
║  │          │   10249   │ TOMSP    │ 1863.40│ 2026-01-05       │   ║
║  │          │   10250   │ HANAR    │ 1552.60│ 2026-01-08       │   ║
║  └──────────┴───────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Installing SSMS

```
1. Download from: https://aka.ms/ssmsfullsetup
2. Run the installer (no options — just click Install)
3. ~800 MB download, ~2 GB installed
4. Launch: Start Menu → "SQL Server Management Studio"
5. Connect: Server = localhost (or .\SQLEXPRESS for Express)
```

### SSMS Must-Know Keyboard Shortcuts ⚡

| Shortcut | Action | When to Use |
|----------|--------|-------------|
| `F5` | Execute query | Run your SQL |
| `Ctrl + E` | Execute query (alternative) | Same as F5 |
| `Ctrl + L` | Show estimated execution plan | Before running — preview the plan |
| `Ctrl + M` | Include actual execution plan | Toggle ON then press F5 |
| `Ctrl + R` | Toggle results pane | Hide/show results |
| `Ctrl + K, C` | Comment selected lines | Block comment |
| `Ctrl + K, U` | Uncomment selected lines | Block uncomment |
| `Ctrl + Shift + U` | UPPERCASE selected text | Format SQL |
| `Ctrl + Shift + L` | lowercase selected text | Format SQL |
| `Ctrl + N` | New query window | Start fresh |
| `Ctrl + T` | Results to text (not grid) | For copying output |
| `Ctrl + D` | Results to grid (default) | Normal view |
| `Alt + F1` | sp_help on selected text | Quick object info |
| `Ctrl + Space` | IntelliSense auto-complete | Code completion |

> 💡 **Pro Tip:** Select a table name and press **Alt+F1** — instantly see its columns, data types, constraints, and indexes. This alone saves hours.

### SSMS Power Features You Should Know

```
┌──────────────────────────────────────────────────────────────┐
│                  SSMS POWER FEATURES                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  📊 EXECUTION PLAN VIEWER                                    │
│     Right-click → Display Estimated/Actual Execution Plan    │
│     See exactly HOW SQL Server runs your query               │
│                                                               │
│  🔍 ACTIVITY MONITOR                                         │
│     Right-click server → Activity Monitor                    │
│     See: Active queries, waits, I/O, CPU in real time        │
│                                                               │
│  📝 TEMPLATE EXPLORER                                         │
│     View → Template Explorer                                  │
│     Pre-built templates for CREATE TABLE, procedures, etc.   │
│                                                               │
│  📦 GENERATE SCRIPTS                                          │
│     Right-click DB → Tasks → Generate Scripts                │
│     Export entire schema as SQL (great for version control)   │
│                                                               │
│  🔄 IMPORT/EXPORT WIZARD                                      │
│     Right-click DB → Tasks → Import/Export Data              │
│     Move data between SQL Server, Excel, CSV, Access, etc.   │
│                                                               │
│  🗺️ DATABASE DIAGRAMS                                        │
│     Right-click Database Diagrams → New Diagram              │
│     Visual ER diagram of your tables & relationships         │
│                                                               │
│  📈 REPORTS                                                    │
│     Right-click server → Reports → Standard Reports          │
│     Built-in reports: disk usage, index stats, memory usage  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 4. Azure Data Studio (ADS) — The Modern Alternative

**Azure Data Studio** is Microsoft's newer, cross-platform, lightweight tool built on VS Code.

```
╔══════════════════════════════════════════════════════════════════╗
║              SSMS vs AZURE DATA STUDIO                           ║
║                                                                  ║
║   ┌────────────────────┐    ┌────────────────────┐              ║
║   │       SSMS         │    │  Azure Data Studio  │              ║
║   ├────────────────────┤    ├────────────────────┤              ║
║   │ Windows ONLY       │    │ Windows/Mac/Linux  │              ║
║   │ Heavy (~2 GB)      │    │ Light (~500 MB)    │              ║
║   │ Full admin features│    │ Query-focused       │              ║
║   │ Object Explorer    │    │ Notebooks! 📓      │              ║
║   │ Agent management   │    │ Extensions          │              ║
║   │ Profiler           │    │ Git integration     │              ║
║   │ Maintenance plans  │    │ Beautiful dashboards│              ║
║   │ SSIS/SSRS designer │    │ PostgreSQL support  │              ║
║   │ Import/Export       │    │ Terminal built-in   │              ║
║   │ Legacy but powerful│    │ Modern & extensible │              ║
║   └────────────────────┘    └────────────────────┘              ║
║                                                                  ║
║   When to use SSMS:                                              ║
║   • DBA tasks (maintenance, Agent, security, SSIS)              ║
║   • Execution plan analysis (deeper visual tools)               ║
║   • Legacy management features                                   ║
║                                                                  ║
║   When to use Azure Data Studio:                                ║
║   • Writing & testing queries                                    ║
║   • Cross-platform development (Mac/Linux)                      ║
║   • Jupyter-style SQL Notebooks                                  ║
║   • Dashboard visualizations                                     ║
║   • PostgreSQL alongside SQL Server                             ║
║                                                                  ║
║   💡 Best practice: Install BOTH!                                ║
╚══════════════════════════════════════════════════════════════════╝
```

### Installing Azure Data Studio

```
1. Download: https://aka.ms/azuredatastudio
2. Available for: Windows, macOS, Linux
3. Install like any app (tiny, quick)
4. Launch → New Connection → Enter server details
```

### Azure Data Studio Killer Features

```
┌──────────────────────────────────────────────────────────┐
│  📓 SQL NOTEBOOKS                                        │
│     • Mix SQL queries, results, and markdown notes       │
│     • Perfect for documentation, tutorials, runbooks     │
│     • Share with team as .ipynb files                    │
│                                                          │
│  📊 BUILT-IN DASHBOARDS                                  │
│     • Server & database dashboards                       │
│     • Customizable widgets (space usage, active queries) │
│     • At-a-glance health monitoring                      │
│                                                          │
│  🧩 EXTENSIONS                                           │
│     • SQL Server Admin Pack                              │
│     • PostgreSQL extension                               │
│     • Schema Compare                                     │
│     • SQL Database Projects                              │
│     • Whois for IP analysis                              │
│                                                          │
│  🎨 MODERN EDITOR                                        │
│     • IntelliSense with snippets                         │
│     • Multi-cursor editing (Alt+Click)                   │
│     • Minimap, bracket matching, code folding            │
│     • Integrated terminal                                │
│     • Dark/Light themes                                  │
└──────────────────────────────────────────────────────────┘
```

---

## ⌨️ 5. sqlcmd — The Command Line Warrior

For scripting, automation, and when you can't use a GUI:

```
┌──────────────────────────────────────────────────────────────┐
│                        sqlcmd                                 │
│           Command-Line Interface for SQL Server               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  # Connect with Windows Authentication                       │
│  sqlcmd -S localhost                                          │
│                                                               │
│  # Connect with SQL Authentication                           │
│  sqlcmd -S localhost -U sa -P "YourPassword"                 │
│                                                               │
│  # Connect to a named instance                               │
│  sqlcmd -S localhost\DEV2026                                  │
│                                                               │
│  # Connect to Azure SQL                                      │
│  sqlcmd -S myserver.database.windows.net -U admin            │
│         -P "Password" -d MyDatabase                           │
│                                                               │
│  # Run a query inline                                        │
│  sqlcmd -S localhost -Q "SELECT @@VERSION"                   │
│                                                               │
│  # Run a SQL script file                                     │
│  sqlcmd -S localhost -i "C:\Scripts\setup.sql"               │
│                                                               │
│  # Output results to a file                                  │
│  sqlcmd -S localhost -Q "SELECT * FROM Products"             │
│         -o "C:\Output\results.txt"                           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### sqlcmd Interactive Mode

```sql
-- Launch interactive mode:
-- > sqlcmd -S localhost

1> SELECT name FROM sys.databases;
2> GO          -- ← GO executes the batch!

-- Output:
-- name
-- ------
-- master
-- tempdb
-- model
-- msdb
-- AdventureWorks

1> :QUIT       -- Exit sqlcmd
```

### Key sqlcmd Flags

| Flag | Purpose | Example |
|------|---------|---------|
| `-S` | Server name | `-S localhost\SQLEXPRESS` |
| `-U` | Username (SQL auth) | `-U sa` |
| `-P` | Password | `-P "MyP@ssw0rd"` |
| `-d` | Database name | `-d AdventureWorks` |
| `-Q` | Execute query and exit | `-Q "SELECT 1"` |
| `-q` | Execute query and stay | `-q "USE master"` |
| `-i` | Input file (run script) | `-i "setup.sql"` |
| `-o` | Output file | `-o "results.txt"` |
| `-E` | Windows authentication | `-E` (no user/pass needed) |
| `-C` | Trust server certificate | Required for some setups |
| `-s` | Column separator | `-s ","` (CSV output) |
| `-W` | Trim trailing spaces | Cleaner output |

> 💡 **Pro Tip:** Use sqlcmd in **CI/CD pipelines**, **Docker containers**, and **automation scripts**. SSMS is for humans; sqlcmd is for robots.

---

## ⚙️ 6. SQL Server Configuration Manager

The behind-the-scenes control panel for SQL Server services and network:

```
╔══════════════════════════════════════════════════════════════════╗
║          SQL SERVER CONFIGURATION MANAGER                        ║
║                                                                  ║
║  ┌───────────────────────────────────────────────────────────┐  ║
║  │  SQL Server Services                                       │  ║
║  │  ├── SQL Server (MSSQLSERVER)      [Running ✅]           │  ║
║  │  ├── SQL Server Agent              [Running ✅]           │  ║
║  │  ├── SQL Server Browser            [Stopped ⛔]           │  ║
║  │  └── SQL Full-Text Filter          [Running ✅]           │  ║
║  │                                                            │  ║
║  │  SQL Server Network Configuration                          │  ║
║  │  ├── Protocols for MSSQLSERVER                            │  ║
║  │  │   ├── Shared Memory             [Enabled ✅]           │  ║
║  │  │   ├── Named Pipes               [Disabled ⛔]          │  ║
║  │  │   └── TCP/IP                    [Enabled ✅]           │  ║
║  │  │       └── IP Addresses                                  │  ║
║  │  │           ├── IP1: 192.168.1.100  Port: 1433           │  ║
║  │  │           ├── IP2: 10.0.0.50      Port: 1433           │  ║
║  │  │           └── IPAll: Port 1433                          │  ║
║  │  │                                                         │  ║
║  │  SQL Native Client Configuration                           │  ║
║  │  └── Client Protocols                                      │  ║
║  └───────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║  How to open:                                                    ║
║  • Start Menu → "SQL Server Configuration Manager"              ║
║  • OR: Computer Management → Services & Applications            ║
║  • OR: Run → SQLServerManager16.msc (for SQL 2022)              ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Common Configuration Tasks

```
┌──────────────────────────────────────────────────────────────┐
│                   COMMON CONFIG TASKS                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  🔌 Enable TCP/IP (for remote connections)                   │
│     1. SQL Server Network Configuration → Protocols          │
│     2. Right-click TCP/IP → Enable                           │
│     3. Properties → IP Addresses → IPAll → Port: 1433       │
│     4. Restart SQL Server service                            │
│                                                               │
│  🔒 Change Service Account                                   │
│     1. SQL Server Services → right-click SQL Server          │
│     2. Properties → Log On tab                               │
│     3. Change to domain account for production               │
│                                                               │
│  🔄 Change Port Number (for security)                        │
│     1. TCP/IP Properties → IP Addresses                      │
│     2. Change IPAll port from 1433 to custom (e.g., 14330)  │
│     3. Restart SQL Server service                            │
│     4. Update firewall rules                                 │
│                                                               │
│  🛡️ Enable Encrypted Connections (SSL/TLS)                   │
│     1. SQL Server properties → Certificate tab               │
│     2. Import SSL certificate                                │
│     3. Set "Force Encryption" = Yes                          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔌 7. Connecting to SQL Server — All Methods

### Connection String Anatomy

```
┌──────────────────────────────────────────────────────────────────┐
│                    CONNECTION STRING                               │
│                                                                   │
│   Server=localhost;                   ← Server name/IP           │
│   Database=AdventureWorks;            ← Database to connect to   │
│   User Id=sa;                         ← SQL login                │
│   Password=MyP@ssw0rd;               ← Password                 │
│   Encrypt=True;                       ← Use SSL/TLS             │
│   TrustServerCertificate=True;        ← Trust self-signed cert  │
│   Connection Timeout=30;              ← Timeout in seconds       │
│   MultipleActiveResultSets=True;      ← MARS support            │
│                                                                   │
│   --- OR with Windows Auth ---                                   │
│   Server=localhost;                                               │
│   Database=AdventureWorks;                                       │
│   Integrated Security=True;           ← Uses Windows login      │
│                                                                   │
│   --- Named Instance ---                                         │
│   Server=localhost\SQLEXPRESS;         ← Backslash + instance   │
│                                                                   │
│   --- Non-default Port ---                                       │
│   Server=localhost,14330;              ← Comma + port number    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Connection Methods Quick Reference

| Method | Connect String / Command | Best For |
|--------|-------------------------|----------|
| **SSMS** | GUI → Server name + Auth | Daily development & admin |
| **Azure Data Studio** | GUI → Connection dialog | Cross-platform, notebooks |
| **sqlcmd** | `sqlcmd -S localhost -U sa -P pass` | Scripts, automation |
| **PowerShell** | `Invoke-Sqlcmd -ServerInstance localhost` | Windows automation |
| **.NET / C#** | `SqlConnection(connString)` | Application code |
| **Python** | `pyodbc.connect(connString)` | Data science, scripting |
| **Node.js** | `mssql.connect(config)` | Web apps |
| **JDBC / Java** | `DriverManager.getConnection(url)` | Enterprise Java apps |

### Connecting from Different Languages

```python
# 🐍 Python (pyodbc)
import pyodbc
conn = pyodbc.connect(
    'DRIVER={ODBC Driver 18 for SQL Server};'
    'SERVER=localhost;'
    'DATABASE=AdventureWorks;'
    'UID=sa;'
    'PWD=YourPassword;'
    'TrustServerCertificate=yes;'
)
cursor = conn.cursor()
cursor.execute("SELECT TOP 5 * FROM Products")
for row in cursor:
    print(row)
```

```csharp
// 🔷 C# (.NET)
using Microsoft.Data.SqlClient;

var connectionString = "Server=localhost;Database=AdventureWorks;"
                     + "User Id=sa;Password=YourPassword;"
                     + "TrustServerCertificate=True;";

using var connection = new SqlConnection(connectionString);
connection.Open();

using var command = new SqlCommand("SELECT TOP 5 * FROM Products", connection);
using var reader = command.ExecuteReader();
while (reader.Read())
{
    Console.WriteLine($"{reader["ProductName"]} - {reader["Price"]}");
}
```

```javascript
// 🟡 Node.js (mssql)
const sql = require('mssql');

const config = {
    server: 'localhost',
    database: 'AdventureWorks',
    user: 'sa',
    password: 'YourPassword',
    options: {
        encrypt: true,
        trustServerCertificate: true
    }
};

const pool = await sql.connect(config);
const result = await pool.request()
    .query('SELECT TOP 5 * FROM Products');
console.table(result.recordset);
```

---

## 🎓 8. First Steps After Installation — Verify Everything

Run these queries immediately after installing SQL Server:

```sql
-- ✅ 1. Check SQL Server version and edition
SELECT @@VERSION;
-- Microsoft SQL Server 2022 (RTM-CU15) - 16.0.4165.4 (X64) Developer Edition

-- ✅ 2. Check server name and instance
SELECT @@SERVERNAME AS ServerName, @@SERVICENAME AS InstanceName;

-- ✅ 3. See all databases
SELECT name, state_desc, compatibility_level, recovery_model_desc
FROM sys.databases
ORDER BY name;

-- ✅ 4. Check authentication mode
SELECT CASE SERVERPROPERTY('IsIntegratedSecurityOnly')
    WHEN 1 THEN 'Windows Authentication Only'
    WHEN 0 THEN 'Mixed Mode (SQL + Windows)'
END AS AuthenticationMode;

-- ✅ 5. Check server configuration
SELECT name, value, value_in_use, description
FROM sys.configurations
WHERE name IN (
    'max server memory (MB)',
    'max degree of parallelism',
    'cost threshold for parallelism',
    'optimize for ad hoc workloads'
)
ORDER BY name;

-- ✅ 6. Install a sample database (AdventureWorks)
-- Download .bak file from:
-- https://github.com/Microsoft/sql-server-samples/releases
-- Then restore:
RESTORE DATABASE AdventureWorks2022
FROM DISK = 'C:\Backups\AdventureWorks2022.bak'
WITH MOVE 'AdventureWorks2022' TO 'C:\Data\AdventureWorks2022.mdf',
     MOVE 'AdventureWorks2022_log' TO 'C:\Data\AdventureWorks2022_log.ldf',
     RECOVERY;
```

### Post-Install Best Practices

```
╔════════════════════════════════════════════════════════════════════╗
║              POST-INSTALL CHECKLIST ✅                             ║
║                                                                    ║
║  □ Set MAX SERVER MEMORY (leave 10-20% for OS)                    ║
║    EXEC sp_configure 'max server memory', 52000;  -- 52 GB       ║
║    RECONFIGURE;                                                    ║
║                                                                    ║
║  □ Set MAXDOP (Max Degree of Parallelism)                         ║
║    Rule: # of cores per NUMA node, max 8                          ║
║    EXEC sp_configure 'max degree of parallelism', 4;              ║
║    RECONFIGURE;                                                    ║
║                                                                    ║
║  □ Set COST THRESHOLD FOR PARALLELISM (default 5 is too low)      ║
║    EXEC sp_configure 'cost threshold for parallelism', 50;        ║
║    RECONFIGURE;                                                    ║
║                                                                    ║
║  □ Enable OPTIMIZE FOR AD HOC WORKLOADS (save plan cache RAM)     ║
║    EXEC sp_configure 'optimize for ad hoc workloads', 1;          ║
║    RECONFIGURE;                                                    ║
║                                                                    ║
║  □ Configure TEMPDB (multiple files = # of CPU cores, max 8)     ║
║                                                                    ║
║  □ Set up BACKUP SCHEDULE (Day 1! Not "I'll do it later")        ║
║                                                                    ║
║  □ Enable QUERY STORE (on by default in SQL 2022)                ║
║    ALTER DATABASE [YourDB] SET QUERY_STORE = ON;                  ║
║                                                                    ║
║  □ Change sa password to something STRONG                         ║
║                                                                    ║
║  □ Disable sa account if not needed                               ║
║    ALTER LOGIN sa DISABLE;                                         ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 🧰 9. Other Useful Tools

| Tool | Purpose | Free? |
|------|---------|-------|
| **SQL Server Profiler** | Trace queries in real-time (legacy — use Extended Events instead) | ✅ (with SSMS) |
| **Extended Events** | Modern lightweight tracing system | ✅ Built-in |
| **Distributed Replay** | Replay production workloads on test servers | ✅ |
| **Database Engine Tuning Advisor (DTA)** | Suggest indexes based on workload | ✅ (with SSMS) |
| **bcp (Bulk Copy)** | Fast bulk import/export of data | ✅ Built-in |
| **SQL Server Data Tools (SSDT)** | Database projects in Visual Studio | ✅ |
| **dbatools (PowerShell)** | 600+ cmdlets for SQL Server management | ✅ Open-source |
| **sp_Blitz (Brent Ozar)** | Health check script — finds problems | ✅ Open-source |
| **sp_WhoIsActive** | See what's running RIGHT NOW | ✅ Open-source |
| **Redgate SQL Toolbelt** | Schema compare, data compare, monitoring | 💰 Paid (industry standard) |

> 💡 **Pro Tip:** Install **dbatools** PowerShell module immediately:
> ```powershell
> Install-Module dbatools -Scope CurrentUser
> # Now you have 600+ cmdlets:
> Get-DbaDatabase -SqlInstance localhost
> Test-DbaMaxMemory -SqlInstance localhost
> ```

---

## 🧠 Quick Recall — Chapter Summary

| Topic | One-Line Summary |
|-------|-----------------|
| Developer Edition | Free, full Enterprise features — best for learning |
| Express Edition | Free, limited (10 GB, 1 GB RAM, 4 cores) — for small apps |
| Standard Edition | Paid, for medium workloads (128 GB RAM, 24 cores) |
| Enterprise Edition | Full power, no limits — for mission-critical production |
| SSMS | The #1 GUI tool — Windows only, full admin + development |
| Azure Data Studio | Modern, cross-platform, notebooks, extensions — query-focused |
| sqlcmd | Command-line SQL client — for automation & scripting |
| Configuration Manager | Manage services, network protocols, ports |
| Docker | Fastest setup — 2 minutes, cross-platform, no installer |
| Connection Strings | `Server=xxx;Database=yyy;User Id=zzz;Password=www;` |
| Post-Install | Set max memory, MAXDOP, cost threshold, backup, Query Store |

---

## ❓ Self-Check Questions

1. Which edition should you use for learning SQL Server? Why?
2. What's the difference between **SSMS** and **Azure Data Studio**?
3. What is `sqlcmd` and when would you use it over SSMS?
4. What is **Mixed Mode** authentication?
5. Why should you ALWAYS set `max server memory`?
6. What is the default TCP port for SQL Server?
7. How do you connect to a **named instance**?
8. What are the first 5 things you should configure after a fresh install?
9. How would you install SQL Server on a Mac?
10. What does the SQL Server Browser service do?

---

## 🎯 Hands-On Challenge

```
□ Install SQL Server Developer Edition (or run via Docker)
□ Install SSMS
□ Install Azure Data Studio
□ Connect to your SQL Server from all 3 tools (SSMS, ADS, sqlcmd)
□ Run SELECT @@VERSION in each tool
□ Download and restore the AdventureWorks sample database
□ Set max server memory to an appropriate value
□ Write a query to list all databases on your server
□ Enable Query Store on AdventureWorks
□ Explore Object Explorer in SSMS — browse tables, views, stored procedures
```

---

> **Next Chapter** → [2C.3 — T-SQL — SQL Server's SQL Dialect](./03-TSQL-Specifics.md)

---

> *"Give me six hours to chop down a tree, and I will spend the first four sharpening the axe."* — Abraham Lincoln
> *...and your axe is SSMS, Azure Data Studio, and sqlcmd.*
