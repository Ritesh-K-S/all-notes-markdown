# ☁️ Chapter 2C.8 — SQL Server on Azure (Azure SQL)

> **Level:** 🟡 Intermediate | 🔥 High Demand
> **Time to Master:** ~5-6 hours
> **Prerequisites:** Chapter 2C.1 (Architecture), Chapter 2C.7 (Administration)

> **"The cloud doesn't eliminate the DBA — it transforms them from a plumber into an architect."**

---

## 📌 What You'll Master

By the end of this chapter, you will:
- Understand **every Azure SQL offering** and when to use each
- Deploy and manage **Azure SQL Database** (PaaS) like a pro
- Choose between **DTU and vCore** pricing without wasting money
- Configure **Elastic Pools** to handle multi-tenant workloads
- Leverage **Serverless** and **Hyperscale** for modern architectures
- Migrate on-premises SQL Server to Azure **without downtime**
- Think in **cloud-native patterns** that save cost and boost performance

---

## 🎯 The Big Picture — Why Azure SQL?

> *"Your CEO just said: Move everything to the cloud. You have 6 months."*

Before we dive into **how**, let's understand **why** companies are migrating:

```
╔══════════════════════════════════════════════════════════════════╗
║           ON-PREMISES vs AZURE SQL — The Real Comparison         ║
╠═════════════════════════╦════════════════════════════════════════╣
║ ON-PREMISES             ║ AZURE SQL                             ║
╠═════════════════════════╬════════════════════════════════════════╣
║                         ║                                        ║
║ Buy hardware upfront    ║ Pay per minute/hour — OpEx not CapEx  ║
║ ($50K-500K servers)     ║ ($5/month for small DBs!)             ║
║                         ║                                        ║
║ Patch OS + SQL yourself ║ Microsoft patches automatically       ║
║ (downtime windows!)     ║ (zero downtime patching)              ║
║                         ║                                        ║
║ Backup to tape/NAS      ║ Automated backups (35-day retention)  ║
║ (hope you tested it)    ║ (tested continuously by Microsoft)    ║
║                         ║                                        ║
║ HA = buy 2x servers +   ║ Built-in HA (99.99% SLA)             ║
║ setup AG (complex)      ║ (no extra setup)                      ║
║                         ║                                        ║
║ Scale up = buy new      ║ Scale up = click a button             ║
║ server (weeks)          ║ (minutes)                             ║
║                         ║                                        ║
║ DR = second data center ║ Geo-replication across regions        ║
║ ($$$$$)                 ║ (built-in, few clicks)                ║
║                         ║                                        ║
║ Security: you manage    ║ Threat detection, vulnerability       ║
║ everything              ║ assessment, auditing — built-in       ║
║                         ║                                        ║
╠═════════════════════════╬════════════════════════════════════════╣
║ STILL NEEDED:           ║                                        ║
║ App development         ║ App development                       ║
║ Schema design           ║ Schema design                         ║
║ Query optimization      ║ Query optimization                    ║
║ Security policies       ║ Security policies                     ║
║                         ║                                        ║
║ 💡 Cloud removes INFRASTRUCTURE work, not DATABASE work.       ║
╚═════════════════════════╩════════════════════════════════════════╝
```

---

## 🗺️ Azure SQL Family — The Complete Map

```
╔══════════════════════════════════════════════════════════════════════╗
║                     AZURE SQL FAMILY (2026)                          ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │                                                              │   ║
║  │  ☁️ AZURE SQL DATABASE                                      │   ║
║  │  (Fully Managed PaaS — Microsoft manages everything)         │   ║
║  │                                                              │   ║
║  │  ┌─────────────┐ ┌─────────────┐ ┌──────────────────────┐  │   ║
║  │  │  Single     │ │  Elastic    │ │  Hyperscale          │  │   ║
║  │  │  Database   │ │  Pool       │ │                      │  │   ║
║  │  │             │ │             │ │  100TB+, instant     │  │   ║
║  │  │  One DB,    │ │  Share      │ │  scale, near-instant │  │   ║
║  │  │  dedicated  │ │  resources  │ │  backup/restore      │  │   ║
║  │  │  resources  │ │  across DBs │ │                      │  │   ║
║  │  └─────────────┘ └─────────────┘ └──────────────────────┘  │   ║
║  │                                                              │   ║
║  │  + SERVERLESS compute tier (auto-pause, pay only when used) │   ║
║  │                                                              │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │                                                              │   ║
║  │  🏢 AZURE SQL MANAGED INSTANCE                              │   ║
║  │  (Near 100% SQL Server compatibility — easiest migration)    │   ║
║  │                                                              │   ║
║  │  • Full SQL Server engine in the cloud                       │   ║
║  │  • SQL Agent, CLR, Cross-DB queries, Linked servers         │   ║
║  │  • Deployed in your own VNet (network isolation)            │   ║
║  │  • Best for: lift-and-shift of existing SQL Server apps      │   ║
║  │                                                              │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐   ║
║  │                                                              │   ║
║  │  🖥️ SQL SERVER ON AZURE VM (IaaS)                           │   ║
║  │  (Full SQL Server on a Virtual Machine — you manage it)      │   ║
║  │                                                              │   ║
║  │  • 100% SQL Server compatibility (it IS SQL Server)          │   ║
║  │  • You manage: OS, patching, backup, HA                      │   ║
║  │  • Best for: features not in PaaS (SSIS, SSRS, SSAS)       │   ║
║  │  • Best for: vendor apps requiring full SQL Server           │   ║
║  │                                                              │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Decision Tree — Which Azure SQL to Choose?

```
                        START HERE
                            │
                  Do you need 100% SQL
                  Server compatibility?
                     /            \
                   YES              NO
                    │                │
            Need OS-level      Can you refactor
            access? SSIS?      queries slightly?
            SSRS? SSAS?           /        \
              /      \          YES          NO
            YES       NO        │            │
             │         │        │     (Full compat needed)
             ▼         ▼        ▼            │
         ┌───────┐ ┌────────┐ ┌───────┐     │
         │SQL on │ │Managed │ │Azure  │     │
         │Azure  │ │Instance│ │SQL DB │     │
         │  VM   │ │        │ │       │     │
         │(IaaS) │ │        │ │       │     │
         └───────┘ └────────┘ └───┬───┘     │
                                  │          │
                          Multi-tenant?      │
                          Many small DBs?    │
                            /       \        │
                          YES        NO      │
                           │          │      │
                    ┌──────▼──┐  ┌────▼───┐ │
                    │ Elastic │  │ Single │ │
                    │  Pool   │  │Database│ │
                    └─────────┘  └────────┘ │
                                            ▼
                                      ┌──────────┐
                                      │ Managed  │
                                      │ Instance │
                                      └──────────┘

  💡 MOST COMMON CHOICES (2026):
  • New cloud-native apps → Azure SQL Database (Single/Serverless)
  • Migrating existing apps → Managed Instance
  • Complex legacy with SSIS/SSRS → SQL on Azure VM
  • SaaS multi-tenant → Elastic Pool
  • 100TB+ analytics → Hyperscale
```

---

## 📊 Azure SQL Database — The Deep Dive

### Purchasing Models: DTU vs vCore

```
╔══════════════════════════════════════════════════════════════════╗
║                DTU vs vCore MODEL                                ║
╠═══════════════════════════════╦══════════════════════════════════╣
║  DTU (Database Transaction    ║  vCore (Virtual Core)            ║
║  Unit) — BUNDLED              ║  — À LA CARTE                   ║
╠═══════════════════════════════╬══════════════════════════════════╣
║                               ║                                  ║
║  DTU = CPU + Memory + IOPS    ║  You pick independently:         ║
║  bundled into a single unit   ║  • Number of vCores (CPU)        ║
║                               ║  • Amount of memory              ║
║  Like buying a combo meal:    ║  • Storage size & speed          ║
║  🍔 + 🍟 + 🥤 = 1 DTU unit  ║                                  ║
║                               ║  Like ordering à la carte:       ║
║  Simple to understand         ║  🍔 separately, 🍟 separately   ║
║  Hard to compare across DBs   ║                                  ║
║                               ║  Easy to compare to on-prem      ║
║  Tiers:                       ║  (8 vCores ≈ 8-core server)     ║
║  • Basic (5 DTU) — $5/mo     ║                                  ║
║  • Standard (10-3000 DTU)     ║  Tiers:                          ║
║  • Premium (125-4000 DTU)     ║  • General Purpose               ║
║                               ║  • Business Critical             ║
║  Best for: simple workloads,  ║  • Hyperscale                    ║
║  predictable resource needs   ║                                  ║
║                               ║  Best for: production, migration ║
║                               ║  Azure Hybrid Benefit eligible!  ║
║                               ║  (use existing SQL licenses)     ║
║                               ║                                  ║
║  💡 Microsoft recommends      ║  ✅ RECOMMENDED for production  ║
║     vCore for new projects    ║                                  ║
║                               ║                                  ║
╚═══════════════════════════════╩══════════════════════════════════╝
```

### Service Tiers — What You Get

```
╔══════════════════════════════════════════════════════════════════════╗
║                  vCore SERVICE TIERS                                  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  GENERAL PURPOSE (Most Common ✅)                           │    ║
║  │                                                             │    ║
║  │  • Remote storage (Azure Premium Storage)                   │    ║
║  │  • Up to 80 vCores, 408GB RAM                               │    ║
║  │  • 5-10ms storage latency                                   │    ║
║  │  • 99.99% SLA                                               │    ║
║  │  • Zone redundant option available                          │    ║
║  │                                                             │    ║
║  │  Best for: Most business applications, web apps, APIs       │    ║
║  │  Analogy: 🚗 A reliable sedan — gets you everywhere       │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  BUSINESS CRITICAL (High Performance)                       │    ║
║  │                                                             │    ║
║  │  • Local SSD storage (super fast!)                          │    ║
║  │  • Up to 128 vCores, 4TB in-memory OLTP                    │    ║
║  │  • 1-2ms storage latency                                   │    ║
║  │  • 99.995% SLA (highest!)                                   │    ║
║  │  • Built-in read replica (free read scale-out!)             │    ║
║  │  • In-Memory OLTP support                                  │    ║
║  │                                                             │    ║
║  │  Best for: OLTP, financial apps, low-latency needs         │    ║
║  │  Analogy: 🏎️ A sports car — maximum performance           │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  ┌─────────────────────────────────────────────────────────────┐    ║
║  │  HYPERSCALE (Massive Scale) 🔥                              │    ║
║  │                                                             │    ║
║  │  • Up to 100TB database size                                │    ║
║  │  • Near-instant backups (regardless of size!)               │    ║
║  │  • Fast restore (minutes, not hours)                        │    ║
║  │  • Up to 4 read replicas (named replicas)                   │    ║
║  │  • Rapidly scale compute up/down                            │    ║
║  │  • Decoupled storage & compute                              │    ║
║  │                                                             │    ║
║  │  Best for: Large databases, unpredictable growth            │    ║
║  │  Analogy: 🚀 A rocket — no limits on scale                │    ║
║  └─────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Hyperscale Architecture — The Game Changer

```
╔══════════════════════════════════════════════════════════════════╗
║                HYPERSCALE ARCHITECTURE                            ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │                    COMPUTE TIER                          │    ║
║  │   ┌──────────┐  ┌──────────┐  ┌──────────┐            │    ║
║  │   │ Primary  │  │ Read     │  │ Read     │            │    ║
║  │   │ Compute  │  │ Replica 1│  │ Replica 2│            │    ║
║  │   │          │  │(named)   │  │(named)   │            │    ║
║  │   └────┬─────┘  └──────────┘  └──────────┘            │    ║
║  │        │                                               │    ║
║  │  ──────┼───── RBPEX (SSD Cache on each node) ──────── │    ║
║  │        │     Caches hot data locally for speed         │    ║
║  └────────┼───────────────────────────────────────────────┘    ║
║           │                                                      ║
║  ┌────────┼───────────────────────────────────────────────┐    ║
║  │        │          PAGE SERVERS                          │    ║
║  │   ┌────▼────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │    ║
║  │   │  Page   │ │  Page   │ │  Page   │ │  Page   │   │    ║
║  │   │Server 1 │ │Server 2 │ │Server 3 │ │Server N │   │    ║
║  │   │(1TB)    │ │(1TB)    │ │(1TB)    │ │(1TB)    │   │    ║
║  │   └─────────┘ └─────────┘ └─────────┘ └─────────┘   │    ║
║  │   Each serves a portion of the database                │    ║
║  └────────────────────────────────────────────────────────┘    ║
║           │                                                      ║
║  ┌────────┼───────────────────────────────────────────────┐    ║
║  │        │          LOG SERVICE                           │    ║
║  │   ┌────▼──────────────────────────────────────────┐   │    ║
║  │   │  Transaction Log → Azure Storage (durable)     │   │    ║
║  │   │  Landing zone → long-term log storage          │   │    ║
║  │   └────────────────────────────────────────────────┘   │    ║
║  └────────────────────────────────────────────────────────┘    ║
║                                                                  ║
║  WHY THIS IS REVOLUTIONARY:                                      ║
║  • Backup of 10TB database: ~seconds (snapshot-based)           ║
║  • Scale compute: ~30 seconds (storage stays put)               ║
║  • Restore 10TB database: ~minutes (point-in-time)              ║
║  • Add read replicas: minutes                                    ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔄 Serverless Compute — Pay Only When Active

```
╔══════════════════════════════════════════════════════════════════╗
║                  SERVERLESS COMPUTE TIER                         ║
║                                                                  ║
║  Traditional (Provisioned):                                      ║
║  ┌──────────────────────────────────────────────────────┐       ║
║  │  ████████████████████████████████████████████████████ │       ║
║  │  You pay 24/7 — even at 3 AM when nobody uses it    │       ║
║  │  $$$$ per month regardless of usage                  │       ║
║  └──────────────────────────────────────────────────────┘       ║
║                                                                  ║
║  Serverless:                                                     ║
║  ┌──────────────────────────────────────────────────────┐       ║
║  │  ██░░░░██████░░░░░░██████████░░░░░░██░░░░░░░░░░░░░░ │       ║
║  │  Active  Sleep Active  Sleep Active Sleep  Sleep     │       ║
║  │  $       $0    $$      $0    $$$    $0     $0        │       ║
║  │                                                      │       ║
║  │  Auto-pause: After 1 hour of inactivity              │       ║
║  │  Auto-resume: First query wakes it up (~1 min)       │       ║
║  │  Auto-scale: 0.5 to 16 vCores based on load          │       ║
║  │                                                      │       ║
║  │  You pay for: compute (per second!) + storage        │       ║
║  └──────────────────────────────────────────────────────┘       ║
║                                                                  ║
║  PERFECT FOR:                                                    ║
║  • Dev/test environments                                        ║
║  • Internal tools used only during business hours               ║
║  • Intermittent workloads (report that runs once daily)         ║
║  • Startups with unpredictable traffic                          ║
║                                                                  ║
║  NOT IDEAL FOR:                                                  ║
║  • 24/7 production (provisioned is cheaper for always-on)       ║
║  • Latency-sensitive (1-min wake-up delay)                      ║
║                                                                  ║
║  💡 SAVINGS: Up to 70% vs provisioned for intermittent use!    ║
╚══════════════════════════════════════════════════════════════════╝
```

```sql
-- Create a Serverless database
-- (via Azure CLI or Azure Portal — not pure T-SQL)

-- Azure CLI example:
-- az sql db create \
--   --resource-group MyRG \
--   --server myserver \
--   --name MyDB \
--   --edition GeneralPurpose \
--   --compute-model Serverless \
--   --auto-pause-delay 60 \      -- Pause after 60 min idle
--   --min-capacity 0.5 \         -- Min 0.5 vCores
--   --max-capacity 4             -- Max 4 vCores
```

---

## 🏊 Elastic Pools — Multi-Tenant Efficiency

### The Problem Elastic Pools Solve

```
WITHOUT ELASTIC POOL (5 databases, each provisioned separately):

  DB1: ████░░░░░░ (uses 40% of its resources)      → paying for 100%
  DB2: ██░░░░░░░░ (uses 20%)                        → paying for 100%
  DB3: ░░░░░░░░░░ (uses 5% — barely used!)          → paying for 100%
  DB4: ██████████ (PEAK — uses 100%!)               → OK
  DB5: ███░░░░░░░ (uses 30%)                        → paying for 100%
  
  Total resources PAID:  500 DTU (100 each × 5)
  Total resources USED:  ~200 DTU average
  WASTE: 60%! 💸💸💸


WITH ELASTIC POOL (same 5 databases, SHARED resources):

  ┌─────────────────────────────────────────────────────┐
  │              ELASTIC POOL (200 eDTU total)           │
  │                                                     │
  │  DB1: ████                                          │
  │  DB2: ██          ← Databases share the pool!      │
  │  DB3: ░           ← Unused resources go to         │
  │  DB4: ██████████  ← whoever needs them NOW         │
  │  DB5: ███                                           │
  │                                                     │
  │  Total: 200 eDTU pool handles all 5 databases      │
  │  Because they DON'T ALL peak at the same time!     │
  │                                                     │
  └─────────────────────────────────────────────────────┘
  
  SAVINGS: 200 eDTU pool vs 500 DTU provisioned = 60% savings!
```

### When to Use Elastic Pools

```
✅ USE ELASTIC POOLS WHEN:
• Multi-tenant SaaS app (1 DB per tenant)
• Many databases with low average utilization
• Databases with unpredictable or variable workloads
• You want to manage cost for 5+ small databases
• Databases that peak at different times

❌ DON'T USE ELASTIC POOLS WHEN:
• Single database that's always busy (use provisioned)
• Databases that ALL peak simultaneously (pool runs out)
• Databases with very different performance needs
• Only 1-2 databases (overhead not worth it)
```

```sql
-- Monitor Elastic Pool usage
SELECT 
    elastic_pool_name,
    avg_cpu_percent,
    avg_data_io_percent,
    avg_log_write_percent,
    max_worker_percent,
    max_session_percent,
    elastic_pool_dtu_limit,
    elastic_pool_storage_limit_mb
FROM sys.elastic_pool_resource_stats
WHERE elastic_pool_name = 'MyPool'
ORDER BY end_time DESC;

-- Find databases that would benefit from a pool
-- (Look for databases with low average but high peak utilization)
SELECT 
    database_name,
    AVG(avg_cpu_percent) AS avg_cpu,
    MAX(avg_cpu_percent) AS peak_cpu,
    AVG(avg_data_io_percent) AS avg_io,
    MAX(avg_data_io_percent) AS peak_io
FROM sys.resource_stats
WHERE start_time > DATEADD(DAY, -14, GETDATE())
GROUP BY database_name
HAVING AVG(avg_cpu_percent) < 40  -- Low average
   AND MAX(avg_cpu_percent) > 80  -- But high peaks
ORDER BY avg_cpu;
```

---

## 🔒 Azure SQL Security — Built-in & Powerful

```
╔══════════════════════════════════════════════════════════════════╗
║              AZURE SQL SECURITY LAYERS                           ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  🌐 NETWORK SECURITY                                           ║
║  ├── Firewall rules (IP-based access control)                   ║
║  ├── Virtual Network service endpoints                          ║
║  ├── Private Link (private IP, no public exposure)              ║
║  └── ⚠️ NEVER leave "Allow Azure services" ON in production   ║
║                                                                  ║
║  🔐 AUTHENTICATION                                             ║
║  ├── Azure AD authentication (recommended ✅)                  ║
║  ├── SQL authentication (legacy, still supported)               ║
║  ├── Multi-Factor Authentication (MFA)                          ║
║  └── Managed Identity (for app-to-DB, no passwords!)           ║
║                                                                  ║
║  🛡️ DATA PROTECTION                                            ║
║  ├── TDE enabled BY DEFAULT (encrypted at rest)                 ║
║  ├── Always Encrypted (client-side encryption)                  ║
║  ├── Dynamic Data Masking                                       ║
║  └── Bring Your Own Key (BYOK) via Azure Key Vault             ║
║                                                                  ║
║  🔍 THREAT DETECTION                                           ║
║  ├── Advanced Threat Protection (ATP)                           ║
║  │   ├── SQL Injection detection                                ║
║  │   ├── Anomalous access patterns                              ║
║  │   ├── Brute force login detection                            ║
║  │   └── Alerts via email to admins                             ║
║  ├── Vulnerability Assessment                                   ║
║  │   └── Scans for misconfigurations, excessive permissions    ║
║  └── Microsoft Defender for SQL                                 ║
║                                                                  ║
║  📋 AUDITING                                                   ║
║  ├── Server-level and database-level auditing                   ║
║  ├── Logs to Azure Storage, Log Analytics, or Event Hub         ║
║  └── Integrates with Microsoft Sentinel (SIEM)                  ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

```sql
-- Azure SQL: Configure firewall (allow specific IP)
-- Via T-SQL:
EXEC sp_set_firewall_rule 
    @name = N'OfficeIP', 
    @start_ip_address = '203.0.113.50', 
    @end_ip_address = '203.0.113.50';

-- Azure SQL: Create Azure AD user
CREATE USER [user@company.com] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [user@company.com];

-- Azure SQL: Create Managed Identity user (for apps)
CREATE USER [MyWebApp] FROM EXTERNAL PROVIDER;  -- App's managed identity name
ALTER ROLE db_datareader ADD MEMBER [MyWebApp];
ALTER ROLE db_datawriter ADD MEMBER [MyWebApp];

-- Connection string using Managed Identity (no passwords!):
-- "Server=myserver.database.windows.net;Database=MyDB;
--  Authentication=Active Directory Managed Identity"
```

---

## 🌍 Geo-Replication & Failover Groups

```
╔══════════════════════════════════════════════════════════════════╗
║         AZURE SQL GEO-REPLICATION & FAILOVER GROUPS              ║
║                                                                  ║
║  ACTIVE GEO-REPLICATION:                                        ║
║  ┌──────────────────┐           ┌──────────────────┐            ║
║  │  PRIMARY          │           │  GEO-SECONDARY   │            ║
║  │  East US          │──async───►│  West Europe      │            ║
║  │  (Read-Write)     │  replica  │  (Read-Only)      │            ║
║  │                    │           │                    │            ║
║  │  myserver.database │           │  myserver-eu.      │            ║
║  │  .windows.net     │           │  database.windows  │            ║
║  └──────────────────┘           │  .net              │            ║
║                                  └──────────────────┘            ║
║  • Up to 4 readable secondaries in different regions            ║
║  • Async replication (typically < 5 seconds lag)                 ║
║  • Manual failover (you decide when to switch)                  ║
║                                                                  ║
║  AUTO-FAILOVER GROUPS (Recommended ✅):                         ║
║  ┌──────────────────┐           ┌──────────────────┐            ║
║  │  PRIMARY          │           │  SECONDARY        │            ║
║  │  East US          │──async───►│  West Europe      │            ║
║  │                    │           │                    │            ║
║  └────────┬─────────┘           └────────┬─────────┘            ║
║           │                               │                      ║
║           └─────────┬─────────────────────┘                      ║
║                     │                                            ║
║           ┌─────────▼──────────┐                                ║
║           │  FAILOVER GROUP    │                                ║
║           │  LISTENER          │ ← Apps connect HERE            ║
║           │                    │                                ║
║           │  R/W: mygroup.     │ → Routes to current PRIMARY   ║
║           │  database.windows  │                                ║
║           │  .net              │                                ║
║           │                    │                                ║
║           │  R/O: mygroup.     │ → Routes to SECONDARY         ║
║           │  secondary.        │                                ║
║           │  database.windows  │                                ║
║           │  .net              │                                ║
║           └────────────────────┘                                ║
║                                                                  ║
║  • AUTOMATIC failover with grace period (1 hour default)        ║
║  • Listener endpoint doesn't change after failover!             ║
║  • No app connection string changes needed!                     ║
║  • Groups multiple databases for consistent failover            ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

```sql
-- Create Failover Group (via Azure CLI)
-- az sql failover-group create \
--   --name mygroup \
--   --partner-server myserver-eu \
--   --resource-group MyRG \
--   --server myserver \
--   --add-db MyDB1 MyDB2 \
--   --failover-policy Automatic \
--   --grace-period 1

-- Check failover group status (via T-SQL on the server)
SELECT 
    fg.name AS failover_group,
    fg.replication_role_desc,
    fg.replication_state_desc,
    d.name AS database_name
FROM sys.dm_geo_replication_link_status ls
JOIN sys.databases d ON ls.database_id = d.database_id
CROSS JOIN (
    SELECT 
        name, 
        replication_role_desc,
        replication_state_desc
    FROM sys.dm_hadr_fabric_continuous_copy_status
) fg;

-- Manual failover (for testing or planned maintenance)
-- az sql failover-group set-primary \
--   --name mygroup \
--   --resource-group MyRG \
--   --server myserver-eu
```

---

## 🚀 Migration to Azure SQL — The Playbook

```
╔══════════════════════════════════════════════════════════════════╗
║             MIGRATION PATH — ON-PREMISES TO AZURE                ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  PHASE 1: ASSESS (What do we have?)                             ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │  Tools:                                                  │    ║
║  │  • Azure Migrate (discovers all SQL servers)             │    ║
║  │  • Data Migration Assistant (DMA)                        │    ║
║  │    → Compatibility issues? Feature parity gaps?         │    ║
║  │  • Azure Database Migration Service (DMS)                │    ║
║  │    → Automated migration with minimal downtime          │    ║
║  │                                                          │    ║
║  │  Key Questions:                                          │    ║
║  │  ✅ Database size and growth rate?                       │    ║
║  │  ✅ Features used? (CLR, Linked Servers, SSIS?)         │    ║
║  │  ✅ Performance requirements (IOPS, latency)?            │    ║
║  │  ✅ Compliance requirements (data residency)?            │    ║
║  │  ✅ Application changes needed?                          │    ║
║  └─────────────────────────────────────────────────────────┘    ║
║                                                                  ║
║  PHASE 2: CHOOSE TARGET                                         ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │                                                          │    ║
║  │  0 compatibility issues → Azure SQL Database ✅         │    ║
║  │  Some compatibility issues → Managed Instance           │    ║
║  │  Many issues / SSIS/SSRS → SQL on Azure VM             │    ║
║  │                                                          │    ║
║  └─────────────────────────────────────────────────────────┘    ║
║                                                                  ║
║  PHASE 3: MIGRATE                                                ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │                                                          │    ║
║  │  OFFLINE MIGRATION (Simple, more downtime)               │    ║
║  │  • Backup on-prem → Restore in Azure                    │    ║
║  │  • BACPAC export → import to Azure SQL DB               │    ║
║  │  • Downtime: hours (depends on DB size)                  │    ║
║  │                                                          │    ║
║  │  ONLINE MIGRATION (Minimal downtime — recommended ✅)    │    ║
║  │  • Azure Database Migration Service (DMS)                │    ║
║  │  • Continuously syncs changes until cutover              │    ║
║  │  • Cutover downtime: minutes                             │    ║
║  │                                                          │    ║
║  │  For Managed Instance:                                    │    ║
║  │  • Log Replay Service (LRS) — restore from Azure Blob   │    ║
║  │  • Managed Instance link (SQL 2022 — near-real-time)    │    ║
║  │                                                          │    ║
║  └─────────────────────────────────────────────────────────┘    ║
║                                                                  ║
║  PHASE 4: OPTIMIZE & CUTOVER                                    ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │  • Validate: Run full test suite against Azure SQL       │    ║
║  │  • Optimize: Right-size compute tier                     │    ║
║  │  • Cutover: Switch DNS/connection strings                │    ║
║  │  • Monitor: Watch performance for 2 weeks                │    ║
║  │  • Decommission: Keep on-prem as rollback for 30 days   │    ║
║  └─────────────────────────────────────────────────────────┘    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 💡 Azure SQL — Key Differences from On-Premises

```
╔══════════════════════════════════════════════════════════════════╗
║  THINGS THAT WORK DIFFERENTLY IN AZURE SQL DATABASE              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ❌ NO SQL Agent (use Azure Automation, Logic Apps, or          ║
║     Elastic Jobs instead)                                        ║
║                                                                  ║
║  ❌ NO cross-database queries (each DB is isolated)             ║
║     Workaround: Elastic Query or Managed Instance                ║
║                                                                  ║
║  ❌ NO USE statement to switch databases                        ║
║     (connect directly to each database)                          ║
║                                                                  ║
║  ❌ NO BULK INSERT from local files                             ║
║     (use Azure Blob Storage + OPENROWSET instead)                ║
║                                                                  ║
║  ❌ NO linked servers                                           ║
║     (use Managed Instance or Elastic Query)                      ║
║                                                                  ║
║  ❌ NO CLR assemblies                                           ║
║     (use Managed Instance for CLR support)                       ║
║                                                                  ║
║  ❌ NO FILESTREAM / FileTable                                   ║
║     (use Azure Blob Storage instead)                              ║
║                                                                  ║
║  ✅ Automatic tuning (auto-create indexes, force good plans!)   ║
║  ✅ Automatic backups (1-35 day retention, PITR)                ║
║  ✅ Built-in HA (no setup needed)                               ║
║  ✅ Intelligent Insights (AI-powered perf analysis)             ║
║  ✅ Elastic Jobs (serverless SQL Agent replacement)             ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 📊 Monitoring Azure SQL — Built-in Intelligence

```sql
-- Azure SQL: Automatic Tuning Recommendations
SELECT 
    reason,
    score,
    JSON_VALUE(details, '$.implementationDetails.script') AS script,
    JSON_VALUE(state, '$.currentValue') AS status
FROM sys.dm_db_tuning_recommendations
ORDER BY score DESC;

-- Azure SQL: Query Performance Insight (via Portal, or T-SQL)
SELECT TOP 10
    qs.query_id,
    qt.query_sql_text,
    rs.avg_duration / 1000000.0 AS avg_duration_seconds,
    rs.avg_cpu_time / 1000000.0 AS avg_cpu_seconds,
    rs.avg_logical_io_reads,
    rs.count_executions
FROM sys.query_store_query_text qt
JOIN sys.query_store_query qs ON qt.query_text_id = qs.query_text_id
JOIN sys.query_store_plan qp ON qs.query_id = qp.query_id
JOIN sys.query_store_runtime_stats rs ON qp.plan_id = rs.plan_id
JOIN sys.query_store_runtime_stats_interval rsi 
    ON rs.runtime_stats_interval_id = rsi.runtime_stats_interval_id
WHERE rsi.start_time > DATEADD(HOUR, -24, GETUTCDATE())
ORDER BY rs.avg_cpu_time DESC;

-- Azure SQL: Resource utilization
SELECT 
    end_time,
    avg_cpu_percent,
    avg_data_io_percent,
    avg_log_write_percent,
    avg_memory_usage_percent,
    max_worker_percent,
    max_session_percent,
    dtu_limit  -- NULL for vCore model
FROM sys.dm_db_resource_stats
ORDER BY end_time DESC;
-- Updated every 15 seconds — real-time monitoring!
```

---

## 💰 Cost Optimization Tips

```
╔══════════════════════════════════════════════════════════════════╗
║            AZURE SQL COST OPTIMIZATION PLAYBOOK                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. 💡 AZURE HYBRID BENEFIT (Save 30-55%!)                     ║
║     If you have existing SQL Server licenses with SA,            ║
║     apply them to Azure SQL → massive discount!                  ║
║                                                                  ║
║  2. 💡 RESERVED CAPACITY (Save 33-65%!)                        ║
║     Commit to 1-year or 3-year → get big discounts              ║
║     Combine with Hybrid Benefit → up to 80% savings!            ║
║                                                                  ║
║  3. 💡 SERVERLESS for dev/test environments                     ║
║     Auto-pause when not in use → pay $0 for compute             ║
║     Only pay for storage during pause                            ║
║                                                                  ║
║  4. 💡 ELASTIC POOLS for multi-tenant                           ║
║     Share resources across databases → 60%+ savings              ║
║                                                                  ║
║  5. 💡 RIGHT-SIZE regularly                                     ║
║     Use Azure Advisor → it tells you if you're over-provisioned ║
║     Scale down during off-hours (via Azure Automation)           ║
║                                                                  ║
║  6. 💡 USE READ REPLICAS instead of scaling up                  ║
║     Route reports/analytics to read-only endpoints               ║
║     Business Critical tier includes 1 free read replica!         ║
║                                                                  ║
║  7. 💡 CLEAN UP unused resources                                ║
║     Delete old test databases                                    ║
║     Remove unused elastic pools                                  ║
║     Check for abandoned resources monthly                        ║
║                                                                  ║
║  EXAMPLE SAVINGS:                                                ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  8 vCore General Purpose:              $750/month        │   ║
║  │  + Azure Hybrid Benefit:               $500/month (-33%) │   ║
║  │  + 3-Year Reserved:                    $280/month (-63%) │   ║
║  │  + Hybrid + Reserved:                  $180/month (-76%) │   ║
║  │                                                          │   ║
║  │  That's $6,840/year saved per database!                   │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🆚 Azure SQL Database vs Managed Instance vs SQL on VM

```
╔═══════════════════════╦═══════════════╦════════════════╦══════════════╗
║ Feature               ║ Azure SQL DB  ║ Managed Inst.  ║ SQL on VM    ║
╠═══════════════════════╬═══════════════╬════════════════╬══════════════╣
║ Management Level      ║ Fully PaaS    ║ Near-PaaS      ║ IaaS (DIY)   ║
║ SQL Compatibility     ║ ~95%          ║ ~99%           ║ 100%         ║
║ Max DB Size           ║ 100TB (Hyper) ║ 16TB           ║ 256TB        ║
║ Cross-DB Queries      ║ ❌ (Elastic) ║ ✅             ║ ✅           ║
║ SQL Agent             ║ ❌           ║ ✅             ║ ✅           ║
║ CLR                   ║ ❌           ║ ✅             ║ ✅           ║
║ Linked Servers        ║ ❌           ║ ✅             ║ ✅           ║
║ SSIS/SSRS/SSAS        ║ ❌           ║ ❌             ║ ✅           ║
║ VNet Integration      ║ Private Link  ║ Native VNet    ║ Full VNet    ║
║ Automated Backup      ║ ✅ Built-in  ║ ✅ Built-in   ║ ❌ Manual    ║
║ Automated Patching    ║ ✅           ║ ✅             ║ Optional     ║
║ Auto-Tuning           ║ ✅           ║ ❌             ║ ❌           ║
║ Serverless Option     ║ ✅           ║ ❌             ║ ❌           ║
║ Elastic Pools         ║ ✅           ║ ✅             ║ ❌           ║
║ Pricing Start         ║ ~$5/mo       ║ ~$350/mo       ║ ~$150/mo     ║
╠═══════════════════════╬═══════════════╬════════════════╬══════════════╣
║ BEST FOR              ║ New cloud-    ║ Migrating      ║ Full SQL     ║
║                       ║ native apps   ║ existing apps  ║ Server needs ║
╚═══════════════════════╩═══════════════╩════════════════╩══════════════╝
```

---

## 🧪 Interview Questions — Azure SQL

### Beginner Level

```
Q1: What are the three Azure SQL deployment options?
A:  1. Azure SQL Database (PaaS — fully managed, per-database)
    2. Azure SQL Managed Instance (PaaS — near-full SQL Server compat)
    3. SQL Server on Azure VM (IaaS — full SQL Server, you manage OS)

Q2: What is the difference between DTU and vCore pricing models?
A:  DTU: Bundled resource unit (CPU + Memory + IOPS combined).
    Simple but hard to compare to on-prem.
    vCore: À la carte — you pick CPU cores, memory, storage 
    independently. Easier to map from on-prem. Supports Azure 
    Hybrid Benefit (bring your own license).

Q3: What is the Serverless compute tier?
A:  Auto-scales vCores based on workload demand and auto-pauses 
    after a configurable idle period. You pay per second of 
    compute used + storage. Best for intermittent, unpredictable 
    workloads. Saves up to 70% vs provisioned.
```

### Advanced Level

```
Q4: You're migrating a 500GB SQL Server database with CLR 
    assemblies and cross-database queries. Which Azure SQL 
    option do you choose?
A:  Azure SQL Managed Instance. Reasons:
    • CLR support ✅ (Azure SQL DB doesn't support CLR)
    • Cross-database queries ✅ (Azure SQL DB doesn't support)
    • Near 100% SQL Server compatibility
    • Deployed in a VNet for network isolation
    • Automated backups and patching
    If SSIS/SSRS is also needed → SQL on Azure VM.

Q5: How do you achieve zero-downtime migration to Azure SQL?
A:  Use Azure Database Migration Service (DMS) in online mode:
    1. Assess compatibility with DMA
    2. Create target Azure SQL database
    3. Set up DMS to continuously replicate changes
    4. DMS does initial full load + ongoing CDC
    5. When ready, perform a brief cutover (minutes):
       - Stop writes to source
       - Wait for DMS to catch up
       - Switch connection strings to Azure
       - Verify and go live
    Total downtime: <5 minutes.

Q6: A customer's Azure SQL Database costs $2,000/month.
    How would you reduce costs?
A:  1. Check Azure Advisor for right-sizing recommendations
    2. Apply Azure Hybrid Benefit (if they have SA licenses) → -33%
    3. Purchase 3-year reserved capacity → additional -45%
    4. Route read workloads to free read replica (Business Critical)
    5. Use Serverless for dev/test environments
    6. Consider Elastic Pool if they have many small databases
    7. Archive old data to Azure Blob Storage (cheaper)
    8. Schedule compute scaling (smaller tier during off-hours)
    Potential savings: 60-80% → $400-800/month instead of $2,000
```

---

## 🗺️ Azure SQL Quick Reference

```
╔══════════════════════════════════════════════════════════════════╗
║                 AZURE SQL CHEAT SHEET                            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  CONNECTION STRING:                                              ║
║  Server=myserver.database.windows.net;Database=MyDB;             ║
║  User ID=myuser;Password=*****;Encrypt=True;                    ║
║  TrustServerCertificate=False;                                   ║
║                                                                  ║
║  FAILOVER GROUP ENDPOINT:                                       ║
║  R/W: mygroup.database.windows.net                               ║
║  R/O: mygroup.secondary.database.windows.net                     ║
║                                                                  ║
║  MAX SIZES:                                                      ║
║  Azure SQL DB (GP): 4TB   │  Hyperscale: 100TB                  ║
║  Managed Instance: 16TB   │  SQL VM: 256TB                      ║
║                                                                  ║
║  SLAs:                                                           ║
║  General Purpose: 99.99%  │  Business Critical: 99.995%         ║
║  Zone Redundant: 99.995%  │  SQL VM: you build your own SLA    ║
║                                                                  ║
║  BACKUP RETENTION:                                               ║
║  Default: 7 days PITR     │  Max: 35 days PITR                  ║
║  Long-term: up to 10 years (LTR policy)                         ║
║                                                                  ║
║  KEY URLS:                                                       ║
║  Portal: portal.azure.com                                        ║
║  Pricing: azure.microsoft.com/pricing/details/azure-sql-database ║
║  Docs: learn.microsoft.com/azure/azure-sql                      ║
║                                                                  ║
║  KEY TOOLS:                                                      ║
║  • Azure Data Studio (cross-platform, modern)                   ║
║  • SSMS (full-featured, Windows only)                            ║
║  • Azure CLI (az sql ...)                                       ║
║  • Azure PowerShell (Az.Sql module)                              ║
║  • Terraform / Bicep (Infrastructure as Code)                   ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🎯 Chapter Summary

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ✅ Azure SQL has 3 deployment options:                        │
│     • Azure SQL Database (PaaS — fully managed, cheapest)      │
│     • Managed Instance (PaaS — near-full SQL Server compat)    │
│     • SQL on Azure VM (IaaS — full control, most work)         │
│                                                                │
│  ✅ Pricing: vCore model recommended over DTU                  │
│     • General Purpose (most workloads)                         │
│     • Business Critical (low latency, free read replica)       │
│     • Hyperscale (100TB+, instant backup/restore)              │
│                                                                │
│  ✅ Serverless = auto-pause + auto-scale = save 70%            │
│                                                                │
│  ✅ Elastic Pools = share resources across databases           │
│     Perfect for multi-tenant SaaS                              │
│                                                                │
│  ✅ Failover Groups = automatic geo-DR with listener           │
│     No app changes needed after failover!                      │
│                                                                │
│  ✅ Cost optimization: Hybrid Benefit + Reserved + right-size  │
│     Can save 60-80% vs pay-as-you-go pricing                   │
│                                                                │
│  ✅ Migration: Azure DMS for online (near-zero downtime)       │
│     DMA for assessment, DMS for execution                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

> **Congratulations!** You've completed the entire **Microsoft SQL Server** track (Chapter 2C).
> **Next:** Move to [Chapter 2D — MySQL](../05-MySQL/01-MySQL-Architecture.md) 🟡⭐ or explore [PART 3 — NoSQL Databases](../08-NoSQL-Foundations/01-NoSQL-Overview.md) 🟠
