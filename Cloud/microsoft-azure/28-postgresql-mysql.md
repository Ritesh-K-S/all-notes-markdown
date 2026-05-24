# Chapter 28: Azure Database for PostgreSQL & MySQL

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Flexible Server Fundamentals](#part-1-flexible-server-fundamentals)
- [Part 2: Creating PostgreSQL Flexible Server (Portal Walkthrough)](#part-2-creating-postgresql-flexible-server-portal-walkthrough)
- [Part 3: Creating MySQL Flexible Server (Portal Walkthrough)](#part-3-creating-mysql-flexible-server-portal-walkthrough)
- [Part 4: High Availability](#part-4-high-availability)
- [Part 5: Read Replicas](#part-5-read-replicas)
- [Part 6: Backup & Restore](#part-6-backup--restore)
- [Part 7: Server Parameters & Tuning](#part-7-server-parameters--tuning)
- [Part 8: Terraform & Bicep](#part-8-terraform--bicep)
- [Part 9: az CLI Reference](#part-9-az-cli-reference)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Database for PostgreSQL and MySQL are fully managed open-source database services. You get the power of PostgreSQL or MySQL without managing the server, OS, patching, or backups. Azure recommends the "Flexible Server" deployment option, which gives you more control over configuration and better cost options.

```
What you'll learn:
├── Flexible Server Fundamentals
│   ├── What is Flexible Server (managed PostgreSQL/MySQL)
│   ├── Flexible Server vs Single Server (deprecated)
│   └── Compute tiers (Burstable, General Purpose, Memory Optimized)
├── Creating PostgreSQL Flexible Server (Portal)
├── Creating MySQL Flexible Server (Portal)
├── High Availability (zone-redundant, same-zone)
├── Read Replicas (offload read traffic)
├── Backup & Restore (PITR)
├── Server Parameters & Tuning
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Flexible Server Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           FLEXIBLE SERVER CONCEPT                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Flexible Server?                                             │
│ ├── Fully managed PostgreSQL or MySQL in Azure                   │
│ ├── You choose: compute tier, storage size, HA, backup           │
│ ├── Azure handles: OS, patching, backups, monitoring             │
│ ├── Supports latest versions (PostgreSQL 16, MySQL 8.0)         │
│ ├── VNet integration (deploy inside your VNet!)                  │
│ └── Flexible maintenance windows (you pick when patches happen) │
│                                                                       │
│ Compute tiers:                                                       │
│ ┌──────────────────┬────────┬────────┬──────────────────────┐    │
│ │ Tier             │ vCPUs  │ RAM    │ Best for              │    │
│ ├──────────────────┼────────┼────────┼──────────────────────┤    │
│ │ Burstable (B)    │ 1-20   │ 2-80 GB│ Dev/test, low-traffic│    │
│ │ General Purpose  │ 2-96   │ 8-384GB│ Production apps      │    │
│ │ (D series)       │        │        │                      │    │
│ │ Memory Optimized │ 2-96   │16-672GB│ Large databases,     │    │
│ │ (E series)       │        │        │ caching, analytics   │    │
│ └──────────────────┴────────┴────────┴──────────────────────┘    │
│                                                                       │
│ Storage:                                                             │
│ ├── Premium SSD (default) or PremiumV2 SSD                       │
│ ├── Size: 32 GB to 64 TB                                        │
│ ├── IOPS: Scales with storage size (or custom for PremiumV2)   │
│ ├── Auto-grow: Storage grows automatically when near full       │
│ └── ⚡ Storage can only INCREASE, never decrease!                │
│                                                                       │
│ ⚡ AWS equivalent: Amazon RDS for PostgreSQL/MySQL                  │
│ ⚡ GCP equivalent: Cloud SQL for PostgreSQL/MySQL                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating PostgreSQL Flexible Server (Portal Walkthrough)

```
Console → Azure Database for PostgreSQL → Flexible Server → Create

┌─────────────────────────────────────────────────────────────────────┐
│           BASICS                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-database-prod ▼]                               │
│                                                                       │
│ Server name: [psql-myapp-prod]                                     │
│ ⚡ Creates: psql-myapp-prod.postgres.database.azure.com           │
│                                                                       │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ PostgreSQL version: [16 ▼]                                        │
│ Options: 13, 14, 15, 16                                            │
│                                                                       │
│ Workload type:                                                       │
│ ○ Development (Burstable, B1ms — cheapest)                       │
│ ● Production (General Purpose or Memory Optimized)                │
│ ○ Production (Small/Medium-size) (General Purpose)               │
│                                                                       │
│ Compute + storage: [General Purpose D2ds_v5 - 2 vCores, 8 GB ▼]│
│ [Configure server] →                                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Compute tier: General Purpose                               │  │
│ │ Compute size: D2ds_v5 (2 vCores, 8 GB RAM) — ~$120/mo    │  │
│ │                                                              │  │
│ │ Storage size: [128] GB (auto-grow enabled)                  │  │
│ │ Storage IOPS: 3000 (scales with size)                       │  │
│ │                                                              │  │
│ │ High availability:                                           │  │
│ │ ☑ Enable HA                                                 │  │
│ │ ○ Zone redundant (standby in different AZ — recommended)   │  │
│ │ ○ Same zone (standby in same AZ — cheaper)                 │  │
│ │                                                              │  │
│ │ Backup retention: [7] days (1-35 days)                     │  │
│ │ Geo-redundant backup: ☐                                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Authentication ──                                                │
│ Authentication method:                                               │
│ ○ PostgreSQL authentication only                                   │
│ ● Microsoft Entra authentication only (recommended!)              │
│ ○ PostgreSQL and Microsoft Entra authentication                  │
│                                                                       │
│ (If PostgreSQL auth):                                               │
│ Admin username: [pgadmin]                                          │
│ Password: [●●●●●●●●●●●●]                                         │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Connectivity method:                                                 │
│ ○ Public access (allowed IP addresses)                             │
│ ● Private access (VNet integration) ← recommended for prod      │
│                                                                       │
│ (If Private access):                                                │
│ Virtual network: [vnet-prod ▼]                                    │
│ Subnet: [subnet-database (10.0.4.0/24) ▼]                       │
│ ⚡ Server deployed INSIDE your VNet! Most secure option.         │
│   Subnet must be delegated to Microsoft.DBforPostgreSQL           │
│                                                                       │
│ Private DNS zone: [privatelink.postgres.database.azure.com ▼]   │
│ ⚡ Auto-creates DNS record for private access.                    │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating MySQL Flexible Server (Portal Walkthrough)

```
Console → Azure Database for MySQL → Flexible Server → Create

Very similar to PostgreSQL, key differences:

┌─────────────────────────────────────────────────────────────────────┐
│           MYSQL SPECIFIC OPTIONS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Server name: [mysql-myapp-prod]                                    │
│ ⚡ Creates: mysql-myapp-prod.mysql.database.azure.com             │
│                                                                       │
│ MySQL version: [8.0 ▼]                                            │
│ Options: 5.7, 8.0                                                   │
│                                                                       │
│ Everything else is the same structure as PostgreSQL:               │
│ ├── Compute tiers (Burstable / General Purpose / Memory Optimized)│
│ ├── Storage (32 GB - 64 TB, auto-grow)                           │
│ ├── HA (zone-redundant or same-zone)                             │
│ ├── Backup (1-35 days retention)                                 │
│ └── Networking (public or private/VNet)                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: High Availability

```
┌─────────────────────────────────────────────────────────────────────┐
│           HIGH AVAILABILITY                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Zone-Redundant HA (recommended for production):                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Availability Zone 1          Availability Zone 2             │  │
│ │ ┌─────────────────┐         ┌─────────────────┐            │  │
│ │ │ Primary Server  │  sync   │ Standby Server  │            │  │
│ │ │ (active)        │ ──→    │ (passive)       │            │  │
│ │ │ Data disk       │ repl   │ Data disk       │            │  │
│ │ └─────────────────┘         └─────────────────┘            │  │
│ │                                                              │  │
│ │ If AZ 1 fails → auto-failover to AZ 2 (~60-120 seconds)  │  │
│ │ Connection endpoint stays the same!                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Same-Zone HA (cheaper):                                              │
│ ├── Both primary and standby in same AZ                          │
│ ├── Protects against: server failures                            │
│ ├── Does NOT protect against: AZ failures                       │
│ └── Good for: non-critical production, staging                  │
│                                                                       │
│ ⚡ HA doubles the compute cost (2 servers running).               │
│ ⚡ Failover is automatic and transparent to your app.            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Read Replicas

```
┌─────────────────────────────────────────────────────────────────────┐
│           READ REPLICAS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Create read-only copies of your database:                            │
│ ├── Up to 5 read replicas                                         │
│ ├── Asynchronous replication (slight lag)                        │
│ ├── Same region or cross-region                                  │
│ ├── Each replica has its own connection endpoint                │
│ └── Application sends writes to primary, reads to replica       │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Primary: psql-myapp-prod.postgres.database.azure.com       │  │
│ │ ├── Handles: INSERT, UPDATE, DELETE (writes)               │  │
│ │                                                              │  │
│ │ Replica 1: psql-myapp-read1.postgres.database.azure.com    │  │
│ │ ├── Handles: SELECT queries (reads only)                   │  │
│ │                                                              │  │
│ │ Replica 2: psql-myapp-read2.postgres.database.azure.com    │  │
│ │ ├── Handles: Analytics/reporting queries                   │  │
│ │                                                              │  │
│ │ ⚡ Offloads read traffic from primary!                      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Portal → PostgreSQL Server → Replication → Add Replica           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Backup & Restore

```
Automatic backups (built-in):
├── Full backup: Weekly
├── Differential: Multiple times daily
├── Transaction log: Every 5 minutes
├── Retention: 1-35 days (default 7)
├── Geo-redundant backup: Optional (copies to paired region)

Restore:
├── Point-in-time restore (any second within retention)
│   Portal → Server → Restore → Pick date/time
│   Creates a NEW server with restored data
├── Geo-restore (from geo-redundant backup to different region)
└── Restore deleted server (within retention period)
```

---

## Part 7: Server Parameters & Tuning

```
Portal → Server → Server parameters

Common parameters to tune:
├── max_connections: Default varies by SKU (increase if needed)
├── shared_buffers: Memory for caching (PostgreSQL)
├── work_mem: Memory per sort/hash operation
├── maintenance_work_mem: Memory for VACUUM, CREATE INDEX
├── log_min_duration_statement: Log slow queries (-1 = off)
├── innodb_buffer_pool_size: Buffer pool size (MySQL)
└── slow_query_log: Enable slow query logging (MySQL)

⚡ Most parameters can be changed without restart.
⚡ Some require restart (search for "dynamic: false" in portal).
```

---

## Part 8: Terraform & Bicep

### Terraform (PostgreSQL)

```hcl
resource "azurerm_postgresql_flexible_server" "main" {
  name                          = "psql-myapp-prod"
  resource_group_name           = azurerm_resource_group.main.name
  location                      = azurerm_resource_group.main.location
  version                       = "16"
  administrator_login           = "pgadmin"
  administrator_password        = var.pg_password
  zone                          = "1"
  storage_mb                    = 131072  # 128 GB
  sku_name                      = "GP_Standard_D2ds_v5"
  backup_retention_days         = 7
  geo_redundant_backup_enabled  = false

  delegated_subnet_id = azurerm_subnet.database.id
  private_dns_zone_id = azurerm_private_dns_zone.postgres.id

  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }
}

resource "azurerm_postgresql_flexible_server_database" "main" {
  name      = "myappdb"
  server_id = azurerm_postgresql_flexible_server.main.id
  collation = "en_US.utf8"
  charset   = "UTF8"
}
```

### Bicep (PostgreSQL)

```bicep
// PostgreSQL Flexible Server
resource postgresServer 'Microsoft.DBforPostgreSQL/flexibleServers@2023-03-01-preview' = {
  name: 'psql-myapp-prod'
  location: resourceGroup().location
  sku: {
    name: 'Standard_D2ds_v5'
    tier: 'GeneralPurpose'
  }
  properties: {
    version: '16'
    administratorLogin: 'pgadmin'
    administratorLoginPassword: pgPassword
    storage: { storageSizeGB: 128 }
    backup: {
      backupRetentionDays: 7
      geoRedundantBackup: 'Disabled'
    }
    highAvailability: {
      mode: 'ZoneRedundant'
      standbyAvailabilityZone: '2'
    }
    availabilityZone: '1'
  }
}

resource postgresDb 'Microsoft.DBforPostgreSQL/flexibleServers/databases@2023-03-01-preview' = {
  parent: postgresServer
  name: 'myappdb'
  properties: {
    charset: 'UTF8'
    collation: 'en_US.utf8'
  }
}
```

### Bicep (MySQL)

```bicep
// MySQL Flexible Server
resource mysqlServer 'Microsoft.DBforMySQL/flexibleServers@2023-06-30' = {
  name: 'mysql-myapp-prod'
  location: resourceGroup().location
  sku: {
    name: 'Standard_D2ds_v4'
    tier: 'GeneralPurpose'
  }
  properties: {
    version: '8.0.21'
    administratorLogin: 'mysqladmin'
    administratorLoginPassword: mysqlPassword
    storage: { storageSizeGB: 128 }
    backup: {
      backupRetentionDays: 7
      geoRedundantBackup: 'Disabled'
    }
    highAvailability: {
      mode: 'ZoneRedundant'
      standbyAvailabilityZone: '2'
    }
  }
}
```
  --name psql-myapp-prod \
  --resource-group rg-database-prod \
  --location centralindia \
  --admin-user pgadmin \
  --admin-password "YourStr0ngP@ssword!" \
  --sku-name Standard_D2ds_v5 \
  --tier GeneralPurpose \
  --version 16 \
  --storage-size 128 \
  --high-availability ZoneRedundant

# Create MySQL Flexible Server
az mysql flexible-server create \
  --name mysql-myapp-prod \
  --resource-group rg-database-prod \
  --location centralindia \
  --admin-user mysqladmin \
  --admin-password "YourStr0ngP@ssword!" \
  --sku-name Standard_D2ds_v5 \
  --tier GeneralPurpose \
  --version 8.0.21 \
  --storage-size 128

# Create database
az postgres flexible-server db create \
  --server-name psql-myapp-prod \
  --resource-group rg-database-prod \
  --database-name myappdb

# Create read replica
az postgres flexible-server replica create \
  --replica-name psql-myapp-read1 \
  --resource-group rg-database-prod \
  --source-server psql-myapp-prod

# List servers
az postgres flexible-server list --resource-group rg-database-prod --output table

# Update server (scale up)
az postgres flexible-server update \
  --name psql-myapp-prod \
  --resource-group rg-database-prod \
  --sku-name Standard_D4ds_v5

# Restore to point in time
az postgres flexible-server restore \
  --name psql-myapp-restored \
  --resource-group rg-database-prod \
  --source-server psql-myapp-prod \
  --restore-time "2024-01-15T14:30:00Z"

# Delete server
az postgres flexible-server delete \
  --name psql-myapp-prod \
  --resource-group rg-database-prod \
  --yes

# Add firewall rule (public access)
az postgres flexible-server firewall-rule create \
  --name AllowMyIP \
  --server-name psql-myapp-prod \
  --resource-group rg-database-prod \
  --start-ip-address 203.0.113.10 \
  --end-ip-address 203.0.113.10
```

---

## Part 10: Real-World Patterns

```
Pattern 1: Production PostgreSQL
├── Server: psql-myapp-prod (GP D4ds_v5, 4 vCores)
├── HA: Zone-redundant
├── Networking: VNet integration (private)
├── Read replicas: 1 (for reporting queries)
├── Backup: 35 days PITR, geo-redundant
├── Auth: Entra ID (managed identity from App Service)
└── Monitoring: Diagnostic settings → Log Analytics

Pattern 2: Multi-Database Microservices (MySQL)
├── Server: mysql-microservices-prod (GP D8ds_v5)
├── Databases: db-users, db-orders, db-inventory, db-payments
├── Each microservice connects to its own database
├── Shared server = cost efficient (vs separate servers)
├── Connection pooling: PgBouncer (PostgreSQL) or ProxySQL (MySQL)
└── HA: Zone-redundant for production
```

---

## Quick Reference

```
Flexible Server = Recommended deployment (most features, best value)
Single Server = Deprecated (migrate to Flexible Server!)

Tiers: Burstable (dev) → General Purpose (prod) → Memory Optimized (large DB)
Storage: 32 GB - 64 TB, auto-grow, can only increase
HA: Zone-redundant (best) or Same-zone (cheaper)

PostgreSQL versions: 13, 14, 15, 16
MySQL versions: 5.7, 8.0

Backup: Automatic PITR (1-35 days) + optional geo-redundant
Read Replicas: Up to 5 (same or cross-region)
Networking: VNet integration (recommended) or public + firewall

Connection strings:
  PostgreSQL: host=<server>.postgres.database.azure.com dbname=mydb user=pgadmin
  MySQL: Server=<server>.mysql.database.azure.com;Database=mydb;Uid=mysqladmin
```

---

## What's Next?

Next chapter: [Chapter 29: Azure Cosmos DB](29-cosmos-db.md) — A globally distributed, multi-model NoSQL database with single-digit millisecond latency.
