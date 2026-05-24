# Chapter 31: Azure SQL Managed Instance

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Managed Instance Fundamentals](#part-1-managed-instance-fundamentals)
- [Part 2: Creating a Managed Instance (Portal Walkthrough)](#part-2-creating-a-managed-instance-portal-walkthrough)
- [Part 3: Migration from On-Premises](#part-3-migration-from-on-premises)
- [Part 4: Instance Pools](#part-4-instance-pools)
- [Part 5: Link Feature (Hybrid)](#part-5-link-feature-hybrid)
- [Part 6: Terraform & az CLI Reference](#part-6-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure SQL Managed Instance is a fully managed SQL Server instance in the cloud with near 100% compatibility with the SQL Server engine. Unlike Azure SQL Database (which is a single database), Managed Instance gives you an entire SQL Server instance — with features like SQL Agent, cross-database queries, CLR, Service Broker, and more. It's the best option for migrating existing on-premises SQL Server applications to Azure with minimal changes.

```
What you'll learn:
├── Managed Instance Fundamentals
│   ├── What is Managed Instance (full SQL Server instance)
│   ├── Managed Instance vs SQL Database vs SQL on VM
│   └── Pricing & compute tiers
├── Creating a Managed Instance (Portal)
├── Migration from On-Premises (DMS, backup/restore)
├── Instance Pools (share resources)
├── Link Feature (near real-time replication to on-prem)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Managed Instance Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGED INSTANCE CONCEPT                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬──────────────────┬──────────────────────┐    │
│ │ Feature          │ SQL Database     │ Managed Instance      │    │
│ ├──────────────────┼──────────────────┼──────────────────────┤    │
│ │ Scope            │ Single database  │ Full SQL instance    │    │
│ │ Cross-DB queries │ No ❌            │ Yes ✅              │    │
│ │ SQL Agent        │ No ❌            │ Yes ✅              │    │
│ │ CLR              │ No ❌            │ Yes ✅              │    │
│ │ Service Broker   │ No ❌            │ Yes ✅              │    │
│ │ Linked servers   │ No ❌            │ Yes ✅              │    │
│ │ SSRS/SSIS        │ No ❌            │ SSRS partial        │    │
│ │ VNet             │ Optional         │ Required (deployed in│    │
│ │                  │                  │ your VNet!)          │    │
│ │ Migration        │ Schema changes   │ Near 100% compat ✅ │    │
│ │ Max databases    │ 1 per resource   │ 100 per instance    │    │
│ │ Provisioning     │ Minutes          │ Hours (first time)  │    │
│ │ Cost             │ Lower            │ Higher               │    │
│ └──────────────────┴──────────────────┴──────────────────────┘    │
│                                                                       │
│ When to use Managed Instance:                                        │
│ ├── Migrating on-prem SQL Server with minimal code changes       │
│ ├── Need SQL Agent for scheduled jobs                             │
│ ├── Need cross-database queries (multiple DBs on one instance)  │
│ ├── Using CLR assemblies, Service Broker, Linked Servers        │
│ └── Need VNet-native deployment (isolated networking)            │
│                                                                       │
│ Service tiers:                                                       │
│ ├── General Purpose: Remote storage, standard workloads          │
│ │   4-80 vCores, up to 16 TB storage                           │
│ └── Business Critical: Local SSD, built-in read replica         │
│     4-128 vCores, up to 16 TB, lowest latency                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Managed Instance (Portal Walkthrough)

```
Console → SQL managed instances → Create

┌─────────────────────────────────────────────────────────────────────┐
│           BASICS                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-sqlmi-prod ▼]                                  │
│                                                                       │
│ Managed instance name: [sqlmi-myapp-prod]                          │
│ ⚡ Creates: sqlmi-myapp-prod.<dns-zone>.database.windows.net     │
│                                                                       │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Compute + storage:                                                   │
│ Service tier: [General Purpose ▼]                                 │
│ Hardware: [Standard-series (Gen5) ▼]                              │
│ vCores: [8 ▼] (4-80)                                             │
│ Storage: [256] GB (32 GB - 16 TB)                                 │
│                                                                       │
│ ⚡ Provisioning takes 4-6 HOURS for first instance in a subnet!  │
│   Subsequent instances in same subnet are faster (~1.5 hours).  │
│                                                                       │
│ Authentication:                                                      │
│ Admin login: [sqladmin]                                             │
│ Password: [●●●●●●●●●●●●]                                         │
│ Entra admin: [Set admin]                                           │
│                                                                       │
│ ── Networking (required!) ──                                        │
│ Virtual network: [vnet-prod ▼]                                    │
│ Subnet: [subnet-sqlmi ▼] (dedicated subnet for MI!)             │
│ ⚡ Managed Instance MUST be in a dedicated subnet.                │
│   Subnet size: /27 minimum (recommended /26 or larger).          │
│   Cannot share with other resources!                              │
│                                                                       │
│ Public endpoint: ○ Enable  ● Disable                              │
│ ⚡ Private by default. Enable public only if needed.              │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Migration from On-Premises

```
┌─────────────────────────────────────────────────────────────────────┐
│           MIGRATION OPTIONS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Azure Database Migration Service (DMS) — recommended           │
│    ├── Online migration (minimal downtime)                        │
│    ├── Continuous sync until cutover                              │
│    └── Portal → Database Migration Service → Create migration   │
│                                                                       │
│ 2. Native backup and restore                                        │
│    ├── Backup on-prem → upload .bak to blob storage             │
│    ├── RESTORE DATABASE FROM URL = 'https://...'                 │
│    ├── Simple but requires downtime                               │
│    └── Good for smaller databases                                 │
│                                                                       │
│ 3. Managed Instance Link (hybrid, near real-time)                  │
│    ├── Always On availability group between on-prem and MI      │
│    ├── Near real-time replication                                 │
│    ├── Gradual migration (test in Azure while prod runs on-prem)│
│    └── Failover when ready                                       │
│                                                                       │
│ Pre-migration assessment:                                            │
│ ├── Azure Migrate: Database Assessment (checks compatibility)   │
│ ├── Data Migration Assistant (DMA) tool                          │
│ └── Identifies: breaking changes, unsupported features, warnings│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Instance Pools

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTANCE POOLS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance Pools = Share compute resources across multiple MIs       │
│                                                                       │
│ ┌───────────────────────────────────────────────────────────────┐ │
│ │ Instance Pool (24 vCores, Gen5)                               │ │
│ │ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐          │ │
│ │ │ MI-1 (4 vCores)│ │ MI-2 (8 vCores)│ │ MI-3 (4 vCores)│      │ │
│ │ │ App A DBs    │ │ App B DBs    │ │ App C DBs    │          │ │
│ │ └──────────────┘ └──────────────┘ └──────────────┘          │ │
│ │ Available: 8 vCores for more instances                       │ │
│ └───────────────────────────────────────────────────────────────┘ │
│                                                                       │
│ Benefits:                                                            │
│ ├── Pre-provision compute for faster deployment (~5 min vs hours)│
│ ├── Cost-effective for multiple small instances                   │
│ ├── Good for: ISVs, multi-tenant applications                   │
│ ├── Each MI is isolated (separate security, backups, etc.)      │
│ └── Share the vCore pool — no waste of unused compute           │
│                                                                       │
│ Create:                                                              │
│ Console → SQL instance pools → [+ Create]                        │
│ ├── Name, Region, VNet/Subnet                                    │
│ ├── Hardware generation (Gen5)                                   │
│ ├── vCores (total for the pool)                                  │
│ └── Then create MIs inside the pool                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Link Feature (Hybrid)

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGED INSTANCE LINK                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Real-time data replication: SQL Server ←→ Managed Instance        │
│ Based on Always On distributed availability groups                 │
│                                                                       │
│ ┌──────────────────┐           ┌──────────────────────┐           │
│ │ On-Premises       │    Link   │ Managed Instance     │           │
│ │ SQL Server        │ ←──────→ │ (Azure)              │           │
│ │ ├── Production DB │  real-   │ ├── Replica DB       │           │
│ │ │   (read/write)  │  time    │ │   (read-only)      │           │
│ │ └── Data synced   │  sync    │ └── Ready for        │           │
│ │    continuously    │          │    failover           │           │
│ └──────────────────┘           └──────────────────────┘           │
│                                                                       │
│ Use cases:                                                           │
│ ├── Hybrid disaster recovery (MI as DR target)                  │
│ ├── Real-time reporting (offload reads to Azure)                │
│ ├── Gradual migration (test in Azure, cutover when ready)       │
│ ├── Cloud-only backups (backup MI, free up on-prem storage)    │
│ └── Dev/test in cloud using production data                     │
│                                                                       │
│ Requirements:                                                        │
│ ├── SQL Server 2016 SP3+ (on-premises)                          │
│ ├── Network connectivity (VPN/ExpressRoute to Azure VNet)       │
│ └── Windows Server Failover Clustering not required              │
│                                                                       │
│ Setup: SQL Server Management Studio (SSMS) wizard                 │
│ ├── Right-click database → Tasks → Azure SQL Managed Instance  │
│ │   Link                                                          │
│ ├── Configure endpoint, certificates                             │
│ └── Start replication → Monitor in SSMS or Portal               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Managing Managed Instance (Portal)

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING MANAGED INSTANCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal → SQL managed instances → [instance-name]                  │
│                                                                       │
│ Overview:                                                            │
│ ├── Status: Ready / Updating / Provisioning                      │
│ ├── Host: sqlmi-myapp-prod.xxxx.database.windows.net            │
│ ├── vCores: 8                                                     │
│ ├── Storage: 256 GB (used: 45 GB)                                │
│ ├── Service tier: General Purpose                                │
│ └── Subnet: subnet-sqlmi / vnet-prod                             │
│                                                                       │
│ Scaling (Compute + storage):                                        │
│ ├── Change vCores: [8 ▼] → [16]                                │
│ │   ⚡ Scaling takes ~30 min - 2 hours                           │
│ │   ⚡ Brief connectivity loss during final failover step       │
│ ├── Change storage: [256 GB] → [512 GB]                         │
│ │   ⚡ Storage can only increase, never decrease               │
│ └── Change tier: General Purpose ↔ Business Critical           │
│     ⚡ Cross-tier migration takes several hours                 │
│                                                                       │
│ Databases:                                                           │
│ ├── View all databases hosted on this instance                  │
│ ├── Create new database                                          │
│ ├── Restore from backup                                          │
│ └── Up to 100 databases per instance                             │
│                                                                       │
│ Failover groups:                                                     │
│ ├── Pair with another MI in a different region                  │
│ ├── Automatic failover if primary region goes down              │
│ ├── Read/write listener DNS: fg-myapp.database.windows.net     │
│ └── Read-only listener: fg-myapp.secondary.database.windows.net│
│                                                                       │
│ Backups:                                                             │
│ ├── Automated backups (7-35 day retention)                      │
│ ├── Long-term retention (LTR): Weekly, monthly, yearly         │
│ ├── Point-in-time restore (any minute within retention)        │
│ └── Portal → Backups → Configure policies                     │
│                                                                       │
│ Security:                                                            │
│ ├── Transparent Data Encryption (TDE): On by default            │
│ ├── Advanced Threat Protection: Detect SQL injection, anomalies │
│ ├── Vulnerability Assessment: Scan for security weaknesses     │
│ └── Auditing: Log all database activities                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Deleting Managed Instance

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETING MANAGED INSTANCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal:                                                              │
│ SQL managed instances → [instance] → [Delete]                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ⚠️  Delete managed instance "sqlmi-myapp-prod"?             │  │
│ │                                                              │  │
│ │ This will permanently delete:                               │  │
│ │ ├── The managed instance                                    │  │
│ │ ├── ALL databases on the instance                           │  │
│ │ └── All automated backups                                   │  │
│ │                                                              │  │
│ │ Type instance name to confirm: [sqlmi-myapp-prod]           │  │
│ │                                                              │  │
│ │ [Delete] [Cancel]                                            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Deletion takes ~30 min to 2 hours (cluster teardown).         │
│ ⚡ LTR backups are NOT deleted (stored separately).              │
│ ⚡ If this is the last MI in the subnet, the subnet remains     │
│   but the MI virtual cluster is cleaned up (takes additional time)│
│                                                                       │
│ CLI:                                                                 │
│   az sql mi delete \                                                │
│     --name sqlmi-myapp-prod \                                       │
│     --resource-group rg-sqlmi-prod \                                │
│     --yes                                                            │
│                                                                       │
│ Before deleting, consider:                                          │
│ ├── Export databases (BACPAC) for archival                       │
│ ├── Remove failover groups first                                 │
│ └── Check if MI Link is configured (disconnect first)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & Bicep

### Terraform

```hcl
# Dedicated subnet for Managed Instance
resource "azurerm_subnet" "sqlmi" {
  name                 = "subnet-sqlmi"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.3.0/26"]

  delegation {
    name = "sqlmi-delegation"
    service_delegation {
      name = "Microsoft.Sql/managedInstances"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action",
        "Microsoft.Network/virtualNetworks/subnets/prepareNetworkPolicies/action",
        "Microsoft.Network/virtualNetworks/subnets/unprepareNetworkPolicies/action",
      ]
    }
  }
}

resource "azurerm_mssql_managed_instance" "main" {
  name                         = "sqlmi-myapp-prod"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_password
  license_type                 = "BasePrice"
  sku_name                     = "GP_Gen5"
  vcores                       = 8
  storage_size_in_gb           = 256
  subnet_id                    = azurerm_subnet.sqlmi.id

  tags = {
    environment = "production"
  }
}

# Failover group (for DR)
resource "azurerm_mssql_managed_instance_failover_group" "main" {
  name                        = "fg-myapp"
  location                    = azurerm_mssql_managed_instance.secondary.location
  managed_instance_id         = azurerm_mssql_managed_instance.main.id
  partner_managed_instance_id = azurerm_mssql_managed_instance.secondary.id

  read_write_endpoint_failover_policy {
    mode          = "Automatic"
    grace_minutes = 60
  }
}
```

### Bicep

```bicep
// Managed Instance
resource managedInstance 'Microsoft.Sql/managedInstances@2023-05-01-preview' = {
  name: 'sqlmi-myapp-prod'
  location: resourceGroup().location
  sku: {
    name: 'GP_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 8
  }
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlPassword
    subnetId: subnet.id
    storageSizeInGB: 256
    licenseType: 'BasePrice'
    vCores: 8
    publicDataEndpointEnabled: false
    timezoneId: 'India Standard Time'
    maintenanceConfigurationId: '/subscriptions/${subscription().subscriptionId}/providers/Microsoft.Maintenance/publicMaintenanceConfigurations/SQL_Default'
  }
  tags: {
    environment: 'production'
  }
}

// Database on managed instance
resource database 'Microsoft.Sql/managedInstances/databases@2023-05-01-preview' = {
  parent: managedInstance
  name: 'myapp-db'
  location: resourceGroup().location
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
  }
}

// Output connection info
output fqdn string = managedInstance.properties.fullyQualifiedDomainName
output adminLogin string = managedInstance.properties.administratorLogin
```

---

## Part 9: az CLI Reference

```bash
# ─── CREATE ───

# Create managed instance (takes 4-6 hours!)
az sql mi create \
  --name sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --location centralindia \
  --admin-user sqladmin \
  --admin-password "YourStr0ngP@ssword!" \
  --subnet /subscriptions/<sub-id>/resourceGroups/rg-sqlmi-prod/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/subnet-sqlmi \
  --sku-name GP_Gen5 \
  --capacity 8 \
  --storage 256GB \
  --license-type BasePrice

# ─── MANAGE ───

# List managed instances
az sql mi list --resource-group rg-sqlmi-prod --output table

# Show instance details
az sql mi show --name sqlmi-myapp-prod --resource-group rg-sqlmi-prod

# Scale vCores
az sql mi update \
  --name sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --capacity 16

# Scale storage
az sql mi update \
  --name sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --storage 512GB

# Change service tier
az sql mi update \
  --name sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --edition BusinessCritical \
  --sku-name BC_Gen5

# Enable public endpoint
az sql mi update \
  --name sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --public-data-endpoint-enabled true

# ─── DATABASES ───

# Create database on MI
az sql midb create \
  --managed-instance sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --name myapp-db

# List databases
az sql midb list \
  --managed-instance sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod --output table

# Restore database from point-in-time
az sql midb restore \
  --managed-instance sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --name myapp-db-restored \
  --source-database myapp-db \
  --time "2024-01-15T10:00:00Z"

# Delete database
az sql midb delete \
  --managed-instance sqlmi-myapp-prod \
  --resource-group rg-sqlmi-prod \
  --name myapp-db --yes

# ─── FAILOVER GROUPS ───

# Create failover group
az sql mi-failover-group create \
  --name fg-myapp \
  --resource-group rg-sqlmi-prod \
  --mi sqlmi-myapp-prod \
  --partner-mi sqlmi-myapp-dr \
  --partner-resource-group rg-sqlmi-dr \
  --failover-policy Automatic \
  --grace-period 60

# Initiate failover
az sql mi-failover-group set-primary \
  --name fg-myapp \
  --resource-group rg-sqlmi-dr \
  --mi sqlmi-myapp-dr

# ─── DELETE ───

# Delete managed instance
az sql mi delete --name sqlmi-myapp-prod --resource-group rg-sqlmi-prod --yes

# Delete failover group
az sql mi-failover-group delete \
  --name fg-myapp \
  --resource-group rg-sqlmi-prod \
  --mi sqlmi-myapp-prod
```

---

## Part 10: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: LIFT-AND-SHIFT SQL SERVER MIGRATION                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Company has 20 SQL Server databases on-premises          │
│ with SQL Agent jobs, cross-database queries, CLR assemblies.       │
│                                                                       │
│ Step 1: Run Azure Migrate Assessment                               │
│   └── Identifies 18/20 DBs are compatible (2 need minor fixes)  │
│                                                                       │
│ Step 2: Set up networking (VPN/ExpressRoute to Azure VNet)        │
│                                                                       │
│ Step 3: Create MI in dedicated subnet                              │
│   └── GP_Gen5, 16 vCores, 1TB (matches on-prem specs)          │
│                                                                       │
│ Step 4: Use DMS for online migration (minimal downtime)           │
│   ├── Initial full backup sync                                    │
│   ├── Continuous log shipping (near real-time)                   │
│   └── Cutover during maintenance window (~30 min downtime)      │
│                                                                       │
│ Step 5: Update application connection strings                      │
│   └── Point to sqlmi-myapp-prod.xxxx.database.windows.net       │
│                                                                       │
│ Result: Same SQL Server features, zero infrastructure management! │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: MULTI-REGION DR WITH FAILOVER GROUPS                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┐           ┌──────────────────────┐           │
│ │ Primary (India)   │   Auto    │ Secondary (Southeast │           │
│ │ sqlmi-myapp-prod  │ ←──────→ │ Asia)                │           │
│ │ ├── Read/Write    │ failover │ sqlmi-myapp-dr        │           │
│ │ └── All app traffic│  group  │ ├── Read-only replica │           │
│ │                    │          │ └── Auto-promoted on  │           │
│ │                    │          │    failure             │           │
│ └──────────────────┘           └──────────────────────┘           │
│                                                                       │
│ App connects to: fg-myapp.database.windows.net                    │
│ ├── Automatically routes to primary                              │
│ ├── On primary failure → Routes to secondary (auto-failover)   │
│ ├── Grace period: 60 minutes before auto-failover               │
│ └── RPO: ~5 seconds (near-zero data loss)                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           SQL MANAGED INSTANCE QUICK REFERENCE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Full SQL Server instance in Azure (near 100% compatible)    │
│ vs SQL Database: MI = full instance; SQL DB = single database     │
│                                                                       │
│ Key features over SQL Database:                                     │
│ SQL Agent, cross-DB queries, CLR, Service Broker, Linked Servers │
│                                                                       │
│ Requirements:                                                        │
│ ├── Dedicated VNet subnet (/27 minimum, /26 recommended)        │
│ ├── First provisioning: 4-6 hours (subsequent: ~1.5 hours)     │
│ └── Scaling: 30 min - 2 hours (brief connectivity loss)        │
│                                                                       │
│ Tiers:                                                               │
│ ├── General Purpose: Remote storage, 4-80 vCores               │
│ └── Business Critical: Local SSD, built-in read replica         │
│                                                                       │
│ Migration:                                                           │
│ ├── DMS (online, minimal downtime) — recommended               │
│ ├── Backup/restore (offline, simple)                             │
│ └── MI Link (hybrid, gradual migration)                          │
│                                                                       │
│ DR: Failover groups (auto-failover, ~5s RPO)                     │
│ Max databases: 100 per instance                                    │
│                                                                       │
│ ⚡ AWS equivalent: Amazon RDS Custom for SQL Server               │
│ ⚡ GCP equivalent: Cloud SQL for SQL Server (limited)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Next chapter: [Chapter 32: Azure DevOps - Repos](32-azure-devops-repos.md) — Git repositories, branch policies, and pull request workflows in Azure DevOps.
