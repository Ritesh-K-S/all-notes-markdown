# Chapter 27: Azure SQL Database

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure SQL Fundamentals](#part-1-azure-sql-fundamentals)
- [Part 2: Creating Azure SQL Database (Full Portal Walkthrough)](#part-2-creating-azure-sql-database-full-portal-walkthrough)
- [Part 3: DTU vs vCore Purchasing Models](#part-3-dtu-vs-vcore-purchasing-models)
- [Part 4: Elastic Pools](#part-4-elastic-pools)
- [Part 5: Serverless Compute Tier](#part-5-serverless-compute-tier)
- [Part 6: Geo-Replication & Failover Groups](#part-6-geo-replication--failover-groups)
- [Part 7: Backup & Restore](#part-7-backup--restore)
- [Part 8: Security](#part-8-security)
- [Part 9: Terraform & Bicep](#part-9-terraform--bicep)
- [Part 10: az CLI Reference](#part-10-az-cli-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure SQL Database is a fully managed relational database service based on the latest stable version of Microsoft SQL Server. You don't manage the server, OS, or patching — Azure handles everything. Just create a database, connect, and start running queries.

```
What you'll learn:
├── Azure SQL Fundamentals
│   ├── What is Azure SQL Database
│   ├── Azure SQL family (Database, Managed Instance, VM)
│   ├── Logical server concept
│   └── When to use which Azure SQL option
├── Creating Azure SQL Database (Full Portal Walkthrough)
│   ├── Server (logical server)
│   ├── Database (name, tier, compute)
│   ├── Networking (firewall, private endpoint)
│   └── Security (Azure AD auth, TDE)
├── Purchasing Models (DTU vs vCore)
├── Elastic Pools (share resources across databases)
├── Serverless (auto-pause, pay-per-use)
├── Geo-Replication & Failover Groups
├── Backup & Restore (PITR, LTR)
├── Security (firewall, TDE, auditing, threat detection)
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Azure SQL Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE SQL FAMILY                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure offers THREE ways to run SQL Server:                          │
│                                                                       │
│ ┌────────────────┬──────────────────────────────────────────────┐  │
│ │ Option          │ Description                                  │  │
│ ├────────────────┼──────────────────────────────────────────────┤  │
│ │ Azure SQL      │ Fully managed single database or elastic pool│  │
│ │ Database       │ Best for: New cloud-native applications     │  │
│ │                │ PaaS — no OS/server access                  │  │
│ │                │ Auto-patching, auto-backups, auto-tuning    │  │
│ │                │                                              │  │
│ │ Azure SQL      │ Fully managed SQL Server INSTANCE            │  │
│ │ Managed        │ Best for: Migrating existing SQL Server apps│  │
│ │ Instance       │ Near 100% compatibility with SQL Server     │  │
│ │                │ SQL Agent, cross-database queries, CLR     │  │
│ │                │                                              │  │
│ │ SQL Server on  │ Full SQL Server on Azure VM                  │  │
│ │ Azure VM       │ Best for: Full OS control needed            │  │
│ │                │ IaaS — you manage OS + SQL Server           │  │
│ │                │ Any SQL Server version/edition              │  │
│ └────────────────┴──────────────────────────────────────────────┘  │
│                                                                       │
│ Decision guide:                                                      │
│ ├── New cloud-native app → Azure SQL Database                   │
│ ├── Migrating from on-prem SQL Server → Managed Instance       │
│ ├── Need specific SQL version or OS access → SQL Server on VM  │
│ └── Multiple small databases → Azure SQL Database Elastic Pool │
│                                                                       │
│ Azure SQL Database architecture:                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ Logical Server: sql-myapp-prod.database.windows.net        │  │
│ │ (Not a real server! Just a logical grouping + firewall)    │  │
│ │                                                              │  │
│ │ ├── Database 1: db-users (General Purpose, 4 vCores)      │  │
│ │ ├── Database 2: db-orders (Business Critical, 8 vCores)   │  │
│ │ ├── Database 3: db-analytics (Hyperscale, 16 vCores)      │  │
│ │ │                                                           │  │
│ │ ├── Elastic Pool: pool-microservices (shared resources)   │  │
│ │ │   ├── db-service-a (uses pool resources)               │  │
│ │ │   ├── db-service-b (uses pool resources)               │  │
│ │ │   └── db-service-c (uses pool resources)               │  │
│ │ │                                                           │  │
│ │ Shared settings:                                            │  │
│ │ ├── Firewall rules (IP whitelist)                          │  │
│ │ ├── Azure AD admin                                         │  │
│ │ ├── Audit settings                                         │  │
│ │ └── Threat detection policies                              │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ AWS equivalent: Amazon RDS for SQL Server                        │
│ ⚡ GCP equivalent: Cloud SQL for SQL Server                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating Azure SQL Database (Full Portal Walkthrough)

```
Console → SQL databases → Create

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-database-prod ▼]                               │
│                                                                       │
│ Database name: [db-myapp-prod]                                     │
│ ⚡ Not globally unique (unique within the server).                 │
│                                                                       │
│ Server: [sql-myapp-prod ▼]  [Create new]                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Create SQL server:                                           │  │
│ │                                                              │  │
│ │ Server name: [sql-myapp-prod]                               │  │
│ │ ⚡ Globally unique! Creates: sql-myapp-prod.database.windows.net│
│ │                                                              │  │
│ │ Location: [Central India ▼]                                 │  │
│ │                                                              │  │
│ │ Authentication method:                                       │  │
│ │ ○ Use SQL authentication                                    │  │
│ │ ○ Use Microsoft Entra-only authentication (recommended!)   │  │
│ │ ● Use both SQL and Microsoft Entra authentication          │  │
│ │                                                              │  │
│ │ Server admin login: [sqladmin]                              │  │
│ │ Password: [●●●●●●●●●●●●]                                  │  │
│ │ ⚡ Store this in Key Vault! You'll need it for connections.│  │
│ │                                                              │  │
│ │ Entra admin: [Set admin] → Select a user or group         │  │
│ │ ⚡ Best practice: Set an Entra admin group for access.     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Want to use SQL elastic pool?: ● No  ○ Yes                       │
│                                                                       │
│ Workload environment:                                                │
│ ● Production (higher defaults)                                     │
│ ○ Development (lower cost defaults)                                │
│                                                                       │
│ Compute + storage: [General Purpose - Serverless ▼]              │
│ [Configure database] →                                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Service tier:                                                │  │
│ │                                                              │  │
│ │ ○ General Purpose (most workloads — balanced price/perf)   │  │
│ │   ├── Remote storage (slightly higher latency)             │  │
│ │   ├── 99.99% SLA                                            │  │
│ │   └── Good for: Web apps, APIs, general business apps     │  │
│ │                                                              │  │
│ │ ○ Business Critical (low latency, high availability)       │  │
│ │   ├── Local SSD storage (fastest!)                         │  │
│ │   ├── Built-in read replica (free!)                        │  │
│ │   ├── 99.995% SLA                                          │  │
│ │   └── Good for: OLTP, financial apps, mission-critical    │  │
│ │                                                              │  │
│ │ ○ Hyperscale (massive scale-out)                            │  │
│ │   ├── Up to 100 TB database size                           │  │
│ │   ├── Instant backups (snapshot-based)                     │  │
│ │   ├── Fast scale up/down                                    │  │
│ │   ├── Up to 4 read replicas                                │  │
│ │   └── Good for: Large databases, unpredictable growth     │  │
│ │                                                              │  │
│ │ Compute tier:                                                │  │
│ │ ● Serverless (auto-pause when idle — saves money!)        │  │
│ │   Min vCores: [0.5]  Max vCores: [4]                      │  │
│ │   Auto-pause delay: [60] minutes                           │  │
│ │   ⚡ Auto-pauses after 60 min idle (no charges!)          │  │
│ │   ⚡ ~10 sec cold start when resuming                     │  │
│ │                                                              │  │
│ │ ○ Provisioned (always running — predictable performance)  │  │
│ │   vCores: [4]                                               │  │
│ │   ⚡ Pay whether you use it or not.                       │  │
│ │                                                              │  │
│ │ Data max size: [32] GB (pay for provisioned size)          │  │
│ │                                                              │  │
│ │ Backup storage redundancy:                                   │  │
│ │ ○ Locally-redundant (cheapest)                              │  │
│ │ ● Zone-redundant (production)                               │  │
│ │ ○ Geo-redundant (disaster recovery)                        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: NETWORKING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Connectivity method:                                                 │
│ ○ No access (configure after creation)                             │
│ ● Public endpoint                                                  │
│ ○ Private endpoint                                                 │
│                                                                       │
│ Firewall rules:                                                      │
│ Allow Azure services and resources: ● Yes  ○ No                  │
│ ⚡ Yes = Any Azure service can connect (App Service, Functions)   │
│                                                                       │
│ Add current client IP address: ● Yes  ○ No                       │
│ ⚡ Adds your current IP so you can connect immediately.           │
│                                                                       │
│ ── Private endpoint (if selected) ──                                │
│ [+ Add private endpoint]                                           │
│ Name: pe-sql-myapp                                                  │
│ VNet: vnet-prod, Subnet: subnet-endpoints                         │
│ ⚡ Database accessible only from your VNet (most secure).        │
│                                                                       │
│ [Next: Security >]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: SECURITY                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Enable Microsoft Defender for SQL: ☑ (Start free trial)          │
│ ⚡ Vulnerability assessment + threat detection.                    │
│                                                                       │
│ Ledger: ☐ Configure a ledger database                             │
│ ⚡ Tamper-proof database (blockchain-like audit trail).           │
│                                                                       │
│ Server identity:                                                     │
│ ☑ Enable system assigned managed identity                        │
│                                                                       │
│ Transparent data encryption (TDE):                                   │
│ ● Service-managed key (default — Azure manages encryption key) │
│ ○ Customer-managed key (your key from Key Vault)               │
│ ⚡ TDE encrypts data at rest — always enabled, transparent.     │
│                                                                       │
│ [Next: Additional settings >]                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 4: ADDITIONAL SETTINGS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Data source:                                                         │
│ ● None (empty database)                                             │
│ ○ Sample (AdventureWorksLT — great for testing!)                 │
│ ○ Backup (restore from backup)                                    │
│                                                                       │
│ Collation: [SQL_Latin1_General_CP1_CI_AS]                         │
│ ⚡ Defines sorting/comparison rules. Default is fine for most.   │
│                                                                       │
│ Maintenance window: ○ System default  ● Schedule                  │
│ ⚡ When Azure can apply patches (choose off-peak hours).         │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: DTU vs vCore Purchasing Models

```
┌─────────────────────────────────────────────────────────────────────┐
│           PURCHASING MODELS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ DTU Model (Database Transaction Unit):                               │
│ ├── Bundled measure of CPU + Memory + IO                          │
│ ├── Simple: Pick a DTU level (5, 10, 20, 50, 100, etc.)        │
│ ├── Can't independently scale CPU vs memory                     │
│ ├── Good for: Simple workloads, easy sizing                     │
│ └── Tiers: Basic (5 DTU) → Standard (10-3000) → Premium       │
│                                                                       │
│ vCore Model (recommended):                                           │
│ ├── Choose CPU (vCores) and storage independently                │
│ ├── More flexible and transparent pricing                        │
│ ├── Azure Hybrid Benefit (use existing SQL Server licenses!)   │
│ ├── Reserved capacity (1-3 year discounts)                      │
│ └── Tiers: General Purpose → Business Critical → Hyperscale   │
│                                                                       │
│ ┌─────────────────┬────────────────┬────────────────────────┐    │
│ │ Feature          │ DTU Model      │ vCore Model            │    │
│ ├─────────────────┼────────────────┼────────────────────────┤    │
│ │ Simplicity       │ Simple ✅     │ More options           │    │
│ │ Flexibility      │ Limited       │ Flexible ✅            │    │
│ │ Hybrid Benefit   │ No ❌         │ Yes ✅ (save 55%!)    │    │
│ │ Reserved capacity│ No            │ Yes ✅ (save 30-60%)  │    │
│ │ Serverless       │ No            │ Yes ✅                 │    │
│ │ Hyperscale       │ No            │ Yes ✅                 │    │
│ │ Best for         │ Simple/small  │ Production ✅          │    │
│ └─────────────────┴────────────────┴────────────────────────┘    │
│                                                                       │
│ ⚡ Use vCore model for production — more control and savings.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Elastic Pools

```
┌─────────────────────────────────────────────────────────────────────┐
│           ELASTIC POOLS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Share compute resources across multiple databases!                   │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Elastic Pool: pool-microservices (200 eDTUs or 4 vCores)   │  │
│ │                                                              │  │
│ │ ├── db-service-a (currently using 50 eDTUs)               │  │
│ │ ├── db-service-b (currently using 10 eDTUs)  ← idle      │  │
│ │ ├── db-service-c (currently using 80 eDTUs)               │  │
│ │ └── db-service-d (currently using 30 eDTUs)               │  │
│ │                                                              │  │
│ │ Total used: 170 / 200 eDTUs                                │  │
│ │                                                              │  │
│ │ ⚡ Without pool: 4 databases × 50 eDTUs each = 200 eDTUs  │  │
│ │   With pool: All share 200 eDTUs (most don't peak at once)│  │
│ │   Result: Same performance, lower cost!                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ When to use:                                                         │
│ ├── Multiple databases with varying usage patterns                │
│ ├── Microservices (one DB per service, most idle most of the time)│
│ ├── SaaS applications (one DB per tenant)                        │
│ └── When total peak < sum of individual peaks                    │
│                                                                       │
│ Portal → SQL server → Elastic pools → New pool                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Serverless Compute Tier

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVERLESS                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Auto-scales compute AND auto-pauses when idle!                      │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Configuration:                                               │  │
│ │ Min vCores: 0.5  Max vCores: 4                              │  │
│ │ Auto-pause delay: 60 minutes (pause after 60 min idle)     │  │
│ │                                                              │  │
│ │ Timeline:                                                    │  │
│ │ 9 AM: Queries come in → scales from 0.5 to 2 vCores       │  │
│ │ 12 PM: Peak traffic → scales to 4 vCores                   │  │
│ │ 6 PM: Traffic drops → scales down to 0.5 vCores           │  │
│ │ 7 PM: No queries for 60 min → AUTO-PAUSED (no charges!)  │  │
│ │ 9 AM next day: First query → resumes (~10 sec cold start) │  │
│ │                                                              │  │
│ │ ⚡ You pay per vCore-second + storage.                      │  │
│ │   When paused: pay ONLY for storage (~$5/month for 32 GB).│  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Best for:                                                            │
│ ├── Dev/test databases (save money when not in use)              │
│ ├── Intermittent workloads (reports, batch jobs)                 │
│ ├── Applications with unpredictable usage                        │
│ └── NOT for: Always-on production (use Provisioned instead)     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Geo-Replication & Failover Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│           GEO-REPLICATION & FAILOVER                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Active Geo-Replication:                                              │
│ ├── Create readable secondary in another region                   │
│ ├── Asynchronous replication (slight lag)                        │
│ ├── Up to 4 secondary replicas                                   │
│ ├── Manual failover (you trigger it)                             │
│ └── Good for: Read offloading + disaster recovery               │
│                                                                       │
│ Auto-failover Groups (recommended for DR):                          │
│ ├── Automatic failover to secondary region                       │
│ ├── Single connection endpoint (doesn't change after failover!) │
│ ├── Read-write listener: <group>.database.windows.net           │
│ ├── Read-only listener: <group>.secondary.database.windows.net │
│ ├── Grace period: 1 hour (customizable)                         │
│ └── Best for: Production disaster recovery                      │
│                                                                       │
│ Setup:                                                               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ Primary: sql-myapp-prod (Central India)                     │  │
│ │ ├── db-myapp-prod (read-write)                             │  │
│ │ │                                                           │  │
│ │ │ ──→ Async replication ──→                                │  │
│ │ │                                                           │  │
│ │ Secondary: sql-myapp-dr (West Europe)                      │  │
│ │ ├── db-myapp-prod (read-only replica)                     │  │
│ │                                                              │  │
│ │ Failover Group: fog-myapp                                   │  │
│ │ R/W: fog-myapp.database.windows.net → Primary            │  │
│ │ R/O: fog-myapp.secondary.database.windows.net → Secondary│  │
│ │                                                              │  │
│ │ ⚡ If primary goes down → auto-failover to secondary!     │  │
│ │   Connection strings DON'T change (use failover group DNS)│  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Backup & Restore

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKUP & RESTORE                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automatic backups (built-in, no setup needed!):                     │
│ ├── Full backup: Weekly                                            │
│ ├── Differential backup: Every 12-24 hours                       │
│ ├── Transaction log backup: Every 5-10 minutes                   │
│ └── ⚡ Completely automatic! No action needed.                    │
│                                                                       │
│ Point-in-Time Restore (PITR):                                       │
│ ├── Restore database to ANY second within retention period       │
│ ├── Default retention: 7 days (configurable 1-35 days)          │
│ ├── Portal → SQL Database → Restore → Pick date/time          │
│ ├── Creates a NEW database (doesn't overwrite existing)         │
│ └── ⚡ "Restore my database to yesterday at 3 PM" — done!      │
│                                                                       │
│ Long-Term Retention (LTR):                                          │
│ ├── Keep backups beyond 35 days (up to 10 years!)               │
│ ├── Weekly, monthly, yearly retention policies                   │
│ ├── Required for compliance (keep 7 years of backups)           │
│ └── Portal → SQL server → Backups → Long-term retention       │
│                                                                       │
│ Deleted database recovery:                                           │
│ ├── Deleted database stays recoverable within retention period  │
│ ├── Portal → SQL server → Deleted databases → Restore         │
│ └── ⚡ Safety net for accidental database deletion!              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY FEATURES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Firewall rules:                                                   │
│    Portal → SQL server → Networking → Firewall rules             │
│    ├── Allow specific IP addresses                                │
│    ├── Allow Azure services (checkbox)                            │
│    └── Private endpoint (most secure — VNet only)                │
│                                                                       │
│ 2. Authentication:                                                   │
│    ├── SQL auth (username/password) — legacy                     │
│    ├── Microsoft Entra auth (Azure AD) — recommended!           │
│    │   Use managed identities from App Service/Functions        │
│    └── Both (SQL + Entra) — for transition period               │
│                                                                       │
│ 3. Transparent Data Encryption (TDE):                                │
│    ├── Encrypts data at rest (database files, backups, logs)    │
│    ├── Always enabled by default (cannot disable)                │
│    └── Service-managed key or customer-managed key (CMK)        │
│                                                                       │
│ 4. Auditing:                                                         │
│    Portal → SQL Database → Auditing → Enable                   │
│    ├── Log all database operations                               │
│    ├── Send to: Log Analytics, Storage Account, Event Hub       │
│    └── Track who did what and when                               │
│                                                                       │
│ 5. Advanced Threat Protection:                                       │
│    ├── SQL injection detection                                    │
│    ├── Anomalous database access patterns                        │
│    ├── Brute force login attempts                                │
│    └── Alerts sent to admins                                     │
│                                                                       │
│ 6. Dynamic Data Masking:                                             │
│    ├── Mask sensitive data (SSN, email, credit card)             │
│    ├── Non-privileged users see masked data                     │
│    └── Portal → SQL Database → Dynamic Data Masking            │
│                                                                       │
│ 7. Row-Level Security:                                               │
│    ├── Users see only their own rows                             │
│    └── Defined via T-SQL security policies                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_mssql_server" "main" {
  name                         = "sql-myapp-prod"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_password
  minimum_tls_version          = "1.2"

  azuread_administrator {
    login_username = "AzureAD Admin"
    object_id      = var.aad_admin_object_id
  }
}

resource "azurerm_mssql_database" "main" {
  name         = "db-myapp-prod"
  server_id    = azurerm_mssql_server.main.id
  collation    = "SQL_Latin1_General_CP1_CI_AS"
  max_size_gb  = 32
  sku_name     = "GP_S_Gen5_2"  # General Purpose Serverless, 2 vCores

  short_term_retention_policy {
    retention_days = 7
  }

  long_term_retention_policy {
    weekly_retention  = "P4W"
    monthly_retention = "P12M"
  }
}

resource "azurerm_mssql_firewall_rule" "allow_azure" {
  name             = "AllowAzureServices"
  server_id        = azurerm_mssql_server.main.id
  start_ip_address = "0.0.0.0"
  end_ip_address   = "0.0.0.0"
}
```

### Bicep

```bicep
// SQL Server
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-myapp-prod'
  location: resourceGroup().location
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlPassword
    version: '12.0'
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Enabled'
    administrators: {
      administratorType: 'ActiveDirectory'
      login: 'AzureAD Admin'
      sid: aadAdminObjectId
      tenantId: subscription().tenantId
      azureADOnlyAuthentication: false
    }
  }
}

// SQL Database (General Purpose Serverless)
resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'db-myapp-prod'
  location: resourceGroup().location
  sku: {
    name: 'GP_S_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 2
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: 34359738368 // 32 GB
    autoPauseDelay: 60
    minCapacity: json('0.5')
  }
}

// Firewall rule
resource firewallRule 'Microsoft.Sql/servers/firewallRules@2023-05-01-preview' = {
  parent: sqlServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}
```
# Create SQL Server (logical)
az sql server create \
  --name sql-myapp-prod \
  --resource-group rg-database-prod \
  --location centralindia \
  --admin-user sqladmin \
  --admin-password "YourStr0ngP@ssword!"

# Create database (serverless)
az sql db create \
  --name db-myapp-prod \
  --server sql-myapp-prod \
  --resource-group rg-database-prod \
  --compute-model Serverless \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2 \
  --min-capacity 0.5 \
  --auto-pause-delay 60 \
  --max-size 32GB

# Add firewall rule (your IP)
az sql server firewall-rule create \
  --server sql-myapp-prod \
  --resource-group rg-database-prod \
  --name AllowMyIP \
  --start-ip-address 203.0.113.10 \
  --end-ip-address 203.0.113.10

# List databases
az sql db list --server sql-myapp-prod --resource-group rg-database-prod --output table

# Scale up database
az sql db update \
  --name db-myapp-prod \
  --server sql-myapp-prod \
  --resource-group rg-database-prod \
  --capacity 4

# Create geo-replica
az sql db replica create \
  --name db-myapp-prod \
  --server sql-myapp-prod \
  --resource-group rg-database-prod \
  --partner-server sql-myapp-dr \
  --partner-resource-group rg-database-dr

# Restore to point in time
az sql db restore \
  --dest-name db-myapp-restored \
  --name db-myapp-prod \
  --server sql-myapp-prod \
  --resource-group rg-database-prod \
  --time "2024-01-15T14:30:00Z"

# Delete database
az sql db delete \
  --name db-myapp-prod \
  --server sql-myapp-prod \
  --resource-group rg-database-prod \
  --yes

# Delete server
az sql server delete \
  --name sql-myapp-prod \
  --resource-group rg-database-prod \
  --yes
```

---

## Part 11: Real-World Patterns

```
Pattern 1: Production Web App Database
├── Server: sql-myapp-prod (Entra auth only)
├── Database: db-myapp-prod (General Purpose, 4 vCores, Provisioned)
├── Networking: Private endpoint (no public access)
├── Geo-replication: Failover group to secondary region
├── Backup: PITR 35 days + LTR weekly/monthly/yearly
├── Monitoring: Auditing → Log Analytics
└── Access: App Service managed identity (no passwords!)

Pattern 2: Dev/Test with Cost Savings
├── Server: sql-myapp-dev
├── Database: db-myapp-dev (General Purpose, Serverless)
│   Min: 0.5 vCores, Max: 2 vCores
│   Auto-pause: 60 minutes
│   ⚡ Cost: ~$5/month when idle (storage only!)
├── Networking: Public endpoint + firewall (dev IPs)
├── Backup: PITR 7 days (default)
└── Data: Restore from production backup (sanitized)
```

---

## Quick Reference

```
Azure SQL Database = Fully managed SQL Server PaaS
Logical Server = Firewall + auth container (not a real server)
Elastic Pool = Share resources across multiple databases

Service tiers:
  General Purpose = Standard workloads (remote storage)
  Business Critical = Low latency (local SSD + free read replica)
  Hyperscale = Up to 100 TB, instant backups, 4 read replicas

Compute: Provisioned (always-on) vs Serverless (auto-pause)
Purchasing: vCore (flexible, recommended) vs DTU (simple)

Backup: Automatic (PITR 1-35 days) + Long-Term Retention (10 years)
Geo-Replication: Active geo-rep (manual) vs Failover Group (automatic)
Security: TDE (always on), Entra auth, firewall, auditing, threat protection
```

---

## What's Next?

Next chapter: [Chapter 28: Azure Database for PostgreSQL/MySQL](28-postgresql-mysql.md) — Flexible Server options for open-source databases in Azure.
