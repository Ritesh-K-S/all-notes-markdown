# Chapter 68: Disaster Recovery & BCDR

---

## Table of Contents

- [Overview](#overview)
- [Part 1: DR Fundamentals](#part-1-dr-fundamentals)
- [Part 2: Azure Backup](#part-2-azure-backup)
- [Part 3: Azure Site Recovery (ASR)](#part-3-azure-site-recovery-asr)
- [Part 4: Database DR Strategies](#part-4-database-dr-strategies)
- [Part 5: Multi-Region DR Patterns](#part-5-multi-region-dr-patterns)
- [Part 6: DR Testing & Runbooks](#part-6-dr-testing--runbooks)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)

---

## Overview

Disaster Recovery (DR) and Business Continuity (BCDR) ensure your applications survive regional outages, data center failures, and data loss. This chapter covers Azure's backup and recovery services, DR patterns, and how to plan RPO/RTO.

```
What you'll learn:
├── DR Fundamentals (RPO, RTO, DR tiers)
├── Azure Backup (VMs, databases, files)
├── Azure Site Recovery (VM replication + failover)
├── Database DR Strategies
├── Multi-Region DR Patterns
├── DR Testing & Runbooks
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: DR Fundamentals

```
Key metrics:
├── RPO (Recovery Point Objective):
│   How much data can you afford to lose?
│   RPO = 1 hour → You accept losing up to 1 hour of data
│   RPO = 0 → No data loss (synchronous replication)
│
├── RTO (Recovery Time Objective):
│   How fast must you recover?
│   RTO = 4 hours → System must be back in 4 hours
│   RTO = 0 → Instant failover (active-active)
│
└── Cost: Lower RPO/RTO = Higher cost!

DR tiers:
┌──────────────────┬────────┬────────┬──────────────────┐
│ Tier             │ RPO    │ RTO    │ Azure Solution   │
├──────────────────┼────────┼────────┼──────────────────┤
│ Backup & Restore │ Hours  │ Hours  │ Azure Backup     │
│ Pilot Light      │ Minutes│ Hours  │ Minimal infra in │
│                  │        │        │ DR region (ready) │
│ Warm Standby     │ Minutes│ Minutes│ Scaled-down copy │
│                  │        │        │ in DR region      │
│ Hot Standby /    │ Seconds│ Seconds│ Active-active    │
│ Multi-Site       │ / Zero │ / Zero │ (Front Door + 2  │
│                  │        │        │  full regions)    │
└──────────────────┴────────┴────────┴──────────────────┘

Disaster types:
├── Hardware failure → Availability Zones protect
├── Data center failure → Availability Zones protect
├── Regional outage → Multi-region DR protects
├── Human error (accidental delete) → Backup protects
├── Cyber attack (ransomware) → Backup + immutable storage
└── Data corruption → Point-in-time restore
```

---

## Part 2: Azure Backup

```
Console → Recovery Services vaults → Create

┌─────────────────────────────────────────────────────────────────────┐
│           AZURE BACKUP                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Create Recovery Services Vault:                                      │
│ Name: [rsv-backup-prod]                                             │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ What can you back up:                                                │
│ ├── Azure VMs (entire VM snapshot)                                │
│ ├── Azure SQL Database (automatic, up to 10 years)               │
│ ├── Azure Files (file share snapshots)                            │
│ ├── Azure Blobs (point-in-time restore)                          │
│ ├── Azure Disks                                                    │
│ ├── Azure Database for PostgreSQL                                 │
│ ├── SAP HANA on Azure VMs                                        │
│ └── On-premises (Windows Server, SQL, files via MARS agent)      │
│                                                                       │
│ Backup Policy:                                                       │
│ Vault → Backup policies → [+ Add]                                │
│ ├── Frequency: Daily at 2:00 AM                                  │
│ ├── Retention:                                                    │
│ │   Daily: Keep for [30] days                                   │
│ │   Weekly: Keep Sunday backup for [12] weeks                   │
│ │   Monthly: Keep first Sunday for [12] months                  │
│ │   Yearly: Keep January backup for [3] years                   │
│ └── Geo-redundancy: GRS (replicate backups to paired region)    │
│                                                                       │
│ Enable VM Backup:                                                    │
│ VM → Backup → Recovery Services vault → Select policy → Enable  │
│ OR                                                                   │
│ Vault → Backup → Azure VM → Select VMs → Enable                 │
│                                                                       │
│ Restore:                                                             │
│ Vault → Backup items → Azure VM → [Restore VM]                  │
│ ├── Create new VM (from backup)                                  │
│ ├── Replace existing (swap disks)                                │
│ └── Restore disks only (attach to existing VM)                  │
│                                                                       │
│ Immutable vault (ransomware protection):                           │
│ ├── Vault → Properties → Immutability → Enable                  │
│ ├── Backups cannot be deleted before retention expires           │
│ ├── Even if admin account is compromised                         │
│ └── Multi-user authorization for critical operations             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Pricing:
├── VM backup: ~$5/month (per instance) + storage
├── SQL backup: Included in SQL Database price
├── Storage: ~$0.05/GB/month (GRS)
└── First 500 GB of Azure Files backup free
```

---

## Part 3: Azure Site Recovery (ASR)

```
ASR = Replicate VMs to another region for disaster recovery

Console → Recovery Services vaults → Site Recovery

How it works:
  Primary Region (Central India)          DR Region (South India)
  ┌──────────────────────┐              ┌──────────────────────┐
  │ VM1 [running]        │  ─replicate→ │ VM1 [standby]       │
  │ VM2 [running]        │  ─replicate→ │ VM2 [standby]       │
  │ VM3 [running]        │  ─replicate→ │ VM3 [standby]       │
  └──────────────────────┘              └──────────────────────┘
  
  If primary region fails:
  → Failover → DR VMs start running
  → DNS updated → Traffic goes to DR region
  → When primary recovers → Failback (reprotect + reverse)

Setup:
1. Recovery Services vault in DR region
2. Vault → Site Recovery → Enable replication
3. Select source VMs → Target region → Replication policy
4. Initial replication starts (full copy)
5. Ongoing: Delta replication (changes only, every 5 min)

Replication policy:
├── Recovery point retention: [24] hours
├── App-consistent snapshot: Every [4] hours
├── Crash-consistent: Every [5] minutes
└── Multi-VM consistency: Group VMs that must fail over together

Failover types:
├── Test Failover: Create VMs in DR, no impact on production
│   ⚡ Do this regularly! (monthly recommended)
├── Planned Failover: Graceful, no data loss
│   Primary → Sync → Stop → Failover → Start DR
├── Unplanned Failover: Emergency, possible data loss
│   Primary down → Failover immediately
│   Use latest recovery point
└── Failback: After primary recovers
    Reprotect → Replicate back → Planned failover to primary

RPO: ~5 minutes (delta replication interval)
RTO: ~2-15 minutes (VM boot time in DR region)

Pricing: ~$25/month per protected VM + storage
```

---

## Part 4: Database DR Strategies

```
Azure SQL Database:
├── Active geo-replication
│   ├── Up to 4 readable secondaries in any region
│   ├── RPO: ~5 seconds (async replication)
│   ├── Failover: Manual or auto-failover group
│   └── Auto-failover group: Automatic failover + DNS redirect
│
├── Auto-failover groups:
│   ├── Automatic failover on regional outage
│   ├── DNS endpoint: <group-name>.database.windows.net
│   ├── Same endpoint before/after failover (no app changes!)
│   └── Grace period: [1] hour (avoid flapping)
│
└── Point-in-time restore:
    ├── Restore to any point in retention period (7-35 days)
    ├── Protects against accidental delete/corruption
    └── Creates new database (doesn't overwrite)

Cosmos DB:
├── Multi-region writes: RPO = 0 (strong consistency)
├── Multi-region reads: RPO < 5 minutes
├── Automatic failover: Region priority list
├── Service-managed failover: Azure detects and fails over
└── Zero configuration DR (just enable multi-region)

Azure Cache for Redis:
├── Geo-replication (Premium tier)
├── Active geo-replication (Enterprise tier): Multi-region writes
└── RDB snapshots for point-in-time restore

Storage:
├── GRS: Geo-redundant (replicate to paired region)
├── RA-GRS: Read-access to secondary (read during outage)
├── GZRS: Zone-redundant + geo-redundant
└── RPO: ~15 minutes for geo-replication
```

---

## Part 5: Multi-Region DR Patterns

```
Pattern 1: Active-Passive (most common)
  Primary: Full stack running, serving traffic
  DR: Minimal (scaled down) or replicated via ASR
  Failover: Manual or automatic via Front Door
  Cost: Low (DR is idle/minimal)

Pattern 2: Active-Active
  Region 1: Full stack, serving traffic
  Region 2: Full stack, serving traffic
  Front Door routes based on latency/health
  Cost: 2x infrastructure
  RTO: Near zero (already running)

Pattern 3: Active-Passive with Read Replicas
  Primary: Writes go here
  DR: Read replicas serve read traffic
  Failover: Promote read replica to primary
  Cost: Medium (replicas running but read-only)

Choosing the right pattern:
┌──────────────────┬────────┬────────┬──────────┐
│ Pattern          │ Cost   │ RPO    │ RTO      │
├──────────────────┼────────┼────────┼──────────┤
│ Backup only      │ $      │ Hours  │ Hours    │
│ Active-Passive   │ $$     │ Minutes│ Minutes  │
│ Active-Active    │ $$$    │ ~Zero  │ ~Zero    │
└──────────────────┴────────┴────────┴──────────┘

Front Door configuration for DR:
  Backend pool: 
    Primary: app-prod-centralindia.azurewebsites.net (Priority 1)
    DR: app-prod-southindia.azurewebsites.net (Priority 2)
  Health probe: /health (every 30s)
  When primary fails health check → Traffic routes to DR
```

---

## Part 6: DR Testing & Runbooks

```
DR Testing (do this regularly!):

1. Tabletop Exercise (quarterly):
   ├── Walk through DR plan on paper
   ├── Identify gaps and update runbooks
   └── No actual failover

2. Test Failover (monthly):
   ├── ASR → Test failover (isolated network)
   ├── Verify apps work in DR region
   ├── Clean up test resources
   └── No impact on production

3. Planned Failover Drill (annually):
   ├── Schedule maintenance window
   ├── Fail over to DR region
   ├── Run in DR for hours/days
   ├── Failback to primary
   └── Document lessons learned

DR Runbook template:
├── Incident Detection
│   ├── How do we know there's a disaster?
│   ├── Who gets notified? (PagerDuty, email)
│   └── Decision criteria for failover
│
├── Failover Steps
│   ├── Step 1: Confirm outage (not transient)
│   ├── Step 2: Notify stakeholders
│   ├── Step 3: Initiate database failover
│   ├── Step 4: Initiate VM failover (ASR)
│   ├── Step 5: Verify DR environment
│   ├── Step 6: Update DNS if needed
│   └── Step 7: Confirm service restored
│
├── Communication
│   ├── Internal: Engineering, leadership
│   ├── External: Status page, customer email
│   └── Updates: Every 30 minutes during incident
│
└── Failback Steps
    ├── Step 1: Confirm primary region recovered
    ├── Step 2: Reprotect (reverse replication)
    ├── Step 3: Planned failback during maintenance window
    ├── Step 4: Verify primary
    └── Step 5: Post-incident review
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
# Recovery Services Vault
resource "azurerm_recovery_services_vault" "main" {
  name                = "rsv-backup-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
  soft_delete_enabled = true
}

# VM Backup Policy
resource "azurerm_backup_policy_vm" "daily" {
  name                = "daily-backup"
  resource_group_name = azurerm_resource_group.main.name
  recovery_vault_name = azurerm_recovery_services_vault.main.name

  backup {
    frequency = "Daily"
    time      = "02:00"
  }

  retention_daily {
    count = 30
  }

  retention_weekly {
    count    = 12
    weekdays = ["Sunday"]
  }
}

# Protect VM
resource "azurerm_backup_protected_vm" "vm1" {
  resource_group_name = azurerm_resource_group.main.name
  recovery_vault_name = azurerm_recovery_services_vault.main.name
  source_vm_id        = azurerm_linux_virtual_machine.vm1.id
  backup_policy_id    = azurerm_backup_policy_vm.daily.id
}

# SQL Auto-Failover Group
resource "azurerm_mssql_failover_group" "main" {
  name      = "fog-myapp"
  server_id = azurerm_mssql_server.primary.id
  databases = [azurerm_mssql_database.main.id]

  partner_server {
    id = azurerm_mssql_server.secondary.id
  }

  read_write_endpoint_failover_policy {
    mode          = "Automatic"
    grace_minutes = 60
  }
}
```

### Bicep

```bicep
// Recovery Services Vault
resource vault 'Microsoft.RecoveryServices/vaults@2023-06-01' = {
  name: 'rsv-dr-prod'
  location: resourceGroup().location
  sku: { name: 'RS0', tier: 'Standard' }
  properties: {
    publicNetworkAccess: 'Enabled'
  }
}

// Site Recovery (replication policy)
resource replicationPolicy 'Microsoft.RecoveryServices/vaults/replicationPolicies@2023-06-01' = {
  parent: vault
  name: 'policy-24h-retention'
  properties: {
    providerSpecificInput: {
      instanceType: 'A2A'
      multiVmSyncStatus: 'Enable'
      appConsistentFrequencyInMinutes: 60
      recoveryPointHistory: 1440
    }
  }
}

// SQL Failover Group
resource failoverGroup 'Microsoft.Sql/servers/failoverGroups@2023-05-01-preview' = {
  parent: sqlServerPrimary
  name: 'fg-myapp-dr'
  properties: {
    partnerServers: [{ id: sqlServerSecondary.id }]
    readWriteEndpoint: {
      failoverPolicy: 'Automatic'
      failoverWithDataLossGracePeriodMinutes: 60
    }
    databases: [sqlDatabase.id]
  }
}
```
az backup vault create \
  --name rsv-backup-prod \
  --resource-group rg-dr \
  --location centralindia

# Enable VM backup
az backup protection enable-for-vm \
  --vault-name rsv-backup-prod \
  --resource-group rg-dr \
  --vm myVM \
  --policy-name DefaultPolicy

# Trigger on-demand backup
az backup protection backup-now \
  --vault-name rsv-backup-prod \
  --resource-group rg-dr \
  --container-name myVM \
  --item-name myVM \
  --retain-until 30-12-2024

# List recovery points
az backup recoverypoint list \
  --vault-name rsv-backup-prod \
  --resource-group rg-dr \
  --container-name myVM \
  --item-name myVM -o table

# Restore VM
az backup restore restore-disks \
  --vault-name rsv-backup-prod \
  --resource-group rg-dr \
  --container-name myVM \
  --item-name myVM \
  --rp-name <recovery-point-name> \
  --storage-account strestoreaccount

# Create SQL failover group
az sql failover-group create \
  --name fog-myapp \
  --resource-group rg-dr \
  --server sql-primary \
  --partner-server sql-secondary \
  --partner-resource-group rg-dr-secondary \
  --add-db myDatabase \
  --failover-policy Automatic \
  --grace-period 1

# Manual failover (for testing)
az sql failover-group set-primary \
  --name fog-myapp \
  --resource-group rg-dr-secondary \
  --server sql-secondary

# Delete vault
az backup vault delete --name rsv-backup-prod --resource-group rg-dr --yes
```

---

## Quick Reference

```
DR Fundamentals:
├── RPO: How much data can you lose? (seconds → hours)
├── RTO: How fast must you recover? (seconds → hours)
└── Lower RPO/RTO = Higher cost

Azure Backup:
├── VMs, SQL, Files, Blobs, PostgreSQL
├── Recovery Services Vault
├── Retention: Daily/Weekly/Monthly/Yearly
├── Immutable vault: Ransomware protection
└── ~$5/month per VM + storage

Azure Site Recovery (ASR):
├── Replicate VMs to DR region (every 5 min)
├── Test/Planned/Unplanned failover
├── RPO: ~5 min | RTO: ~2-15 min
└── ~$25/month per protected VM

Database DR:
├── SQL: Auto-failover groups (automatic, same DNS endpoint)
├── Cosmos DB: Multi-region (automatic, built-in)
├── Point-in-time restore: Undo accidental deletes

Patterns:
├── Active-Passive: Low cost, minutes to failover
├── Active-Active: High cost, near-zero downtime
└── Front Door: Automatic health-based routing

⚡ Test your DR plan regularly!
⚡ Automate failover where possible
⚡ Document runbooks for manual steps
```

---

## Congratulations! 🎉

You've completed the entire Azure documentation series — from cloud fundamentals through disaster recovery. You now have comprehensive knowledge of Azure's core services, security, monitoring, analytics, AI, migration, and architecture best practices.

**Go back to the index:** [Chapter 00: Index](00-index.md)
