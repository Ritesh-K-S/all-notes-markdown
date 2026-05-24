# Chapter 62: Azure Migrate

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Migration Fundamentals](#part-1-migration-fundamentals)
- [Part 2: Azure Migrate Hub (Portal Walkthrough)](#part-2-azure-migrate-hub-portal-walkthrough)
- [Part 3: Discovery & Assessment](#part-3-discovery--assessment)
- [Part 4: Server Migration](#part-4-server-migration)
- [Part 5: Database Migration](#part-5-database-migration)
- [Part 6: App Migration](#part-6-app-migration)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Migrate is a centralized hub for discovering, assessing, and migrating on-premises servers, databases, and applications to Azure. It helps you plan your migration and execute it with minimal downtime.

```
What you'll learn:
├── Migration Fundamentals (strategies, phases)
├── Azure Migrate Hub (Portal)
├── Discovery & Assessment (what do you have, what does it cost)
├── Server Migration (VMs → Azure)
├── Database Migration (SQL Server → Azure SQL)
├── App Migration (web apps → App Service)
├── az CLI reference
└── Quick reference
```

---

## Part 1: Migration Fundamentals

```
Migration strategies (The 5 Rs):
├── Rehost ("Lift & Shift")
│   Move VMs as-is to Azure (no code changes)
│   Fastest, lowest risk, but no cloud optimization
│
├── Refactor ("Lift & Reshape")
│   Move to PaaS with minimal changes
│   SQL Server → Azure SQL, App → App Service
│
├── Rearchitect
│   Redesign for cloud-native (containers, serverless)
│   More effort but better scalability/cost
│
├── Rebuild
│   Rewrite from scratch using cloud services
│   Most effort, best cloud optimization
│
└── Replace
    Replace with SaaS solution (e.g., Exchange → Office 365)

Migration phases:
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Discover │ → │ Assess   │ → │ Migrate  │ → │ Optimize │
│          │   │          │   │          │   │          │
│ What do  │   │ Right    │   │ Move     │   │ Right-   │
│ you have?│   │ size,    │   │ workloads│   │ size,    │
│          │   │ cost,    │   │ to Azure │   │ secure,  │
│          │   │ readiness│   │          │   │ monitor  │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
```

---

## Part 2: Azure Migrate Hub (Portal Walkthrough)

```
Console → Azure Migrate

┌─────────────────────────────────────────────────────────────────────┐
│           AZURE MIGRATE HUB                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Migration goals:                                                     │
│ ├── Servers, databases, and web apps                              │
│ ├── SQL Server (databases only)                                   │
│ ├── SAP on Azure                                                  │
│ ├── VDI (Virtual Desktop)                                         │
│ └── Data (large datasets)                                         │
│                                                                       │
│ Tools:                                                               │
│ ├── Discovery and assessment                                      │
│ │   └── Discover servers, assess readiness and cost              │
│ ├── Migration and modernization                                   │
│ │   └── Migrate VMs (agentless or agent-based)                  │
│ ├── Data Migration Assistant (DMA)                                │
│ │   └── Assess SQL Server databases                              │
│ ├── Azure Database Migration Service (DMS)                       │
│ │   └── Migrate databases with minimal downtime                 │
│ ├── Web app migration assistant                                   │
│ │   └── Assess and migrate web apps to App Service              │
│ └── Azure Data Box                                                │
│     └── Ship large datasets physically                           │
│                                                                       │
│ [Create project]                                                    │
│ Project name: [migrate-datacenter-2024]                            │
│ Geography: [India]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Discovery & Assessment

```
Step 1: Deploy Azure Migrate Appliance
├── Download OVA (VMware) or VHD (Hyper-V) from portal
├── Deploy as VM in your on-premises environment
├── Lightweight VM that discovers your servers
├── Agentless: No software installed on target servers

Step 2: Discovery
├── Appliance connects to vCenter / Hyper-V host
├── Discovers all VMs: OS, CPU, memory, disk, network
├── Software inventory: Installed apps, roles, features
├── Dependency analysis: Which servers talk to each other
│   ├── Agentless: Network connections (no agent needed)
│   └── Agent-based: Detailed dependencies (install agent)
└── Discovery runs continuously (updated every 30 min)

Step 3: Assessment
├── Azure Migrate → Assessments → [+ Assess]
├── Assessment types:
│   ├── Azure VM: What VMs do you need in Azure?
│   ├── Azure SQL: Can your SQL migrate to Azure SQL?
│   ├── Azure App Service: Can your apps run on App Service?
│   └── AVD: Virtual desktop assessment
│
├── Output:
│   ├── Readiness: Ready / Ready with conditions / Not ready
│   ├── Size recommendation: Right-sized VM SKU
│   ├── Monthly cost estimate: Pay-as-you-go or reserved
│   ├── Storage recommendation: Standard/Premium SSD
│   └── Dependency map: Visual server relationships
│
├── Sizing criteria:
│   ├── Performance-based: Uses actual CPU/memory/disk utilization
│   └── As on-premises: Matches current specs (may oversize)
│
└── ⚡ Performance-based is recommended (avoid over-provisioning)
```

---

## Part 4: Server Migration

```
Migration methods:
├── Agentless (VMware only):
│   ├── No agent installed on source VMs
│   ├── Uses vCenter for replication
│   ├── Simpler setup, minimal impact on source
│   └── Recommended for VMware environments
│
├── Agent-based:
│   ├── Install Mobility Service agent on each VM
│   ├── Works for VMware, Hyper-V, physical servers, other clouds
│   └── More flexible but requires agent installation
│
└── Hyper-V migration:
    ├── Built-in Hyper-V replication
    └── No agent needed

Migration process:
1. Enable replication
   Azure Migrate → Migration tools → Replicate
   Select VMs → Target VNet/Subnet → Start replication

2. Initial replication
   Full copy of disks to Azure (may take hours/days)

3. Delta replication
   Ongoing sync of changes (continuous)

4. Test migration
   Create test VM in Azure → Validate → Clean up test
   ⚡ Always test before production migration!

5. Production migration
   Schedule maintenance window
   → Stop source VM
   → Final sync
   → Create Azure VM
   → Verify
   → Decommission source

Cutover options:
├── Planned: Schedule, minimal downtime (delta sync)
└── Unplanned: Emergency (may lose recent changes)
```

---

## Part 5: Database Migration

```
Azure Database Migration Service (DMS):

Console → Database Migration Service → Create

Supported migrations:
├── SQL Server → Azure SQL Database (offline/online)
├── SQL Server → Azure SQL Managed Instance (online)
├── SQL Server → SQL Server on Azure VM
├── PostgreSQL → Azure Database for PostgreSQL
├── MySQL → Azure Database for MySQL
├── MongoDB → Azure Cosmos DB
└── Oracle → Azure Database for PostgreSQL

Online vs Offline:
├── Offline: Source unavailable during migration
│   ├── Simpler, lower risk
│   └── Downtime = data transfer time
│
└── Online: Continuous sync, minimal downtime
    ├── Initial load + Change Data Capture (CDC)
    ├── Cutover when caught up
    └── Downtime = cutover time only (minutes)

Migration steps (SQL Server → Azure SQL):
1. Run Data Migration Assistant (DMA) for assessment
   ├── Compatibility issues
   ├── Feature parity gaps
   └── Performance recommendations

2. Create DMS instance
   ├── SKU: Standard (offline) or Premium (online)
   └── Connect to source and target

3. Create migration project
   ├── Source: SQL Server (connection string)
   ├── Target: Azure SQL Database
   ├── Select tables/schemas to migrate
   └── Start migration

4. Monitor & cutover
   ├── Watch replication lag
   ├── When caught up → Cutover
   └── Update application connection strings
```

---

## Part 6: App Migration

```
Azure App Service Migration Assistant:

Tool: https://azure.microsoft.com/en-us/products/app-service/migration-tools

Step 1: Run assessment on your web server
├── Checks IIS/.NET/Java app compatibility
├── Reports: Ready / Ready with warnings / Not ready
├── Common issues:
│   ├── HTTPS bindings → Configure in App Service
│   ├── Windows Authentication → Use Entra ID instead
│   ├── Local file system access → Use Azure Files
│   └── Registry access → Use App Settings
│
Step 2: Migrate
├── Creates App Service Plan + Web App
├── Deploys application code
├── Configures settings
└── Sets up custom domain (optional)

Containerized app migration:
├── Azure Migrate → App containerization
├── Automatically containerizes ASP.NET / Java apps
├── Creates Dockerfile
├── Pushes to ACR
└── Deploys to AKS or App Service (containers)
```

---

## Part 7: az CLI Reference

```bash
# --- Azure Migrate Project ---

# Create migration project
az migrate project create \
  --name migrate-datacenter-2024 \
  --resource-group rg-migrate \
  --location centralindia

# List projects
az migrate project list --resource-group rg-migrate -o table

# Show project details
az migrate project show \
  --name migrate-datacenter-2024 \
  --resource-group rg-migrate

# Delete project
az migrate project delete \
  --name migrate-datacenter-2024 \
  --resource-group rg-migrate --yes

# --- Database Migration Service ---

# Create DMS instance
az dms create \
  --name dms-migration \
  --resource-group rg-migrate \
  --location centralindia \
  --sku-name Premium_4vCores \
  --subnet /subscriptions/<sub-id>/resourceGroups/rg-migrate/providers/Microsoft.Network/virtualNetworks/vnet-migrate/subnets/snet-dms

# List DMS instances
az dms list --resource-group rg-migrate -o table

# Show DMS details
az dms show --name dms-migration --resource-group rg-migrate

# Create migration project (in DMS)
az dms project create \
  --name proj-sql-migration \
  --resource-group rg-migrate \
  --service-name dms-migration \
  --source-platform SQL \
  --target-platform SQLDB

# List migration projects
az dms project list \
  --service-name dms-migration \
  --resource-group rg-migrate -o table

# Create migration task (SQL Server → Azure SQL)
az dms project task create \
  --name task-migrate-orderdb \
  --resource-group rg-migrate \
  --service-name dms-migration \
  --project-name proj-sql-migration \
  --task-type MigrateSqlToSqlDb \
  --source-connection-json @source-conn.json \
  --target-connection-json @target-conn.json \
  --database-options-json @db-options.json

# Check migration task status
az dms project task show \
  --name task-migrate-orderdb \
  --resource-group rg-migrate \
  --service-name dms-migration \
  --project-name proj-sql-migration

# Delete DMS instance
az dms delete --name dms-migration --resource-group rg-migrate --yes

# --- Server Migration (via Azure Migrate) ---
# Note: Server replication/migration is mainly done via the Portal
# The az CLI support is limited. Use Portal for:
# - Appliance setup
# - Discovery
# - Assessment creation
# - Replication and migration
```

---

## Quick Reference

```
Azure Migrate = Central hub for migration to Azure

5 Rs: Rehost | Refactor | Rearchitect | Rebuild | Replace

Phases: Discover → Assess → Migrate → Optimize

Tools:
├── Azure Migrate Appliance: Discover on-prem servers (agentless)
├── Assessment: Readiness, sizing, cost estimation
├── Server Migration: Agentless (VMware) or Agent-based
├── Database Migration Service (DMS): Online (min downtime) or Offline
├── App Migration Assistant: IIS/.NET apps → App Service
└── Data Box: Ship large datasets physically

Server migration: Replicate → Test migrate → Production migrate → Cutover
Database migration: Assess (DMA) → Migrate (DMS) → Cutover

⚡ Always use performance-based sizing (not as-on-premises)
⚡ Always test migration before production cutover
⚡ Online migration = minimal downtime (recommended)
```

---

## What's Next?

Next chapter: [Chapter 63: Azure Arc](63-azure-arc.md) — Manage on-premises, multi-cloud, and edge resources from Azure.
