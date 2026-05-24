# Chapter 24: Azure Files & File Sync

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Files Fundamentals](#part-1-azure-files-fundamentals)
- [Part 2: Creating a File Share (Full Portal Walkthrough)](#part-2-creating-a-file-share-full-portal-walkthrough)
- [Part 3: Connecting to File Shares](#part-3-connecting-to-file-shares)
- [Part 4: Azure File Sync](#part-4-azure-file-sync)
- [Part 5: Snapshots & Backup](#part-5-snapshots--backup)
- [Part 6: Terraform & Bicep](#part-6-terraform--bicep)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Files provides fully managed file shares in the cloud that are accessible via the standard SMB (Server Message Block) and NFS (Network File System) protocols. Think of it as a network drive in the cloud — you can mount it on Windows, Linux, or macOS just like a shared folder on your office network.

```
What you'll learn:
├── Azure Files Fundamentals
│   ├── What is Azure Files (cloud file shares)
│   ├── SMB vs NFS protocols
│   ├── Standard vs Premium tiers
│   └── Azure Files vs Blob Storage vs Managed Disks
├── Creating a File Share (Portal Walkthrough)
├── Connecting from Windows, Linux, macOS
├── Azure File Sync (sync on-prem with cloud)
├── Snapshots & Backup
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Azure Files Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE FILES CONCEPT                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Azure Files?                                                │
│ ├── Cloud-based file shares (like a network drive)               │
│ ├── Access via SMB 3.x (Windows/Linux/macOS) or NFS 4.1 (Linux)│
│ ├── Mount as a drive letter on Windows (Z:\)                    │
│ ├── Mount as a path on Linux (/mnt/azure-share)                │
│ ├── Fully managed (no server to maintain)                       │
│ ├── Multiple VMs/users can access simultaneously                │
│ └── Supports snapshots, soft delete, backup                     │
│                                                                       │
│ Use cases:                                                           │
│ ├── Replace on-premises file servers                             │
│ ├── Shared configuration files across VMs                        │
│ ├── Application logs (centralized, shared location)             │
│ ├── Dev/test environments (shared data)                         │
│ ├── Legacy applications that need file share                     │
│ └── Lift-and-shift applications from on-premises                │
│                                                                       │
│ Tiers:                                                               │
│ ┌──────────────┬───────────────────────────────────────────────┐  │
│ │ Tier         │ Description                                   │  │
│ ├──────────────┼───────────────────────────────────────────────┤  │
│ │ Premium      │ SSD-backed, low latency, high IOPS            │  │
│ │              │ Provisioned model (pay for allocated capacity)│  │
│ │              │ Use: databases, high-performance apps         │  │
│ │              │                                               │  │
│ │ Transaction  │ HDD-backed, pay-per-transaction               │  │
│ │ Optimized    │ Use: general file shares, team shares        │  │
│ │ (Hot)        │                                               │  │
│ │              │                                               │  │
│ │ Cool         │ Lower storage cost, higher transaction cost  │  │
│ │              │ Use: online archive, infrequent access       │  │
│ │              │                                               │  │
│ │ Cold         │ Lowest storage cost, highest transaction cost │  │
│ │              │ Use: backup, rarely accessed data            │  │
│ └──────────────┴───────────────────────────────────────────────┘  │
│                                                                       │
│ Comparison:                                                          │
│ ┌─────────────────┬──────────────┬──────────────┬─────────────┐  │
│ │ Feature          │ Azure Files  │ Blob Storage │ Managed Disk│  │
│ ├─────────────────┼──────────────┼──────────────┼─────────────┤  │
│ │ Protocol         │ SMB/NFS      │ REST API     │ Block device│  │
│ │ Mount as drive   │ Yes ✅       │ No (BlobFuse)│ Yes (1 VM)  │  │
│ │ Multi-VM access  │ Yes ✅       │ Yes (API)    │ No (1 VM)   │  │
│ │ Max size         │ 100 TB       │ 190 TB/blob  │ 64 TB       │  │
│ │ Best for         │ File shares  │ Object data  │ VM disk     │  │
│ └─────────────────┴──────────────┴──────────────┴─────────────┘  │
│                                                                       │
│ ⚡ AWS equivalent: EFS (Elastic File System)                        │
│ ⚡ GCP equivalent: Filestore                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a File Share (Full Portal Walkthrough)

```
Console → Storage Account → File shares → + File share

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE FILE SHARE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [shared-files]                                                │
│ ⚡ Rules: 3-63 chars, lowercase, numbers, hyphens                  │
│                                                                       │
│ Access tier:                                                         │
│ ● Transaction optimized (Hot — general purpose)                   │
│ ○ Cool (infrequent access)                                        │
│ ○ Cold (rare access)                                               │
│ ○ Premium (SSD — only if storage account is Premium File)        │
│                                                                       │
│ Provisioned capacity (Premium only): [100] GiB                    │
│ ⚡ Premium shares: You pay for provisioned capacity, not used.    │
│   IOPS and throughput scale with provisioned size.                │
│                                                                       │
│ Enable backup: ☑                                                   │
│ Backup vault: [vault-prod ▼]                                     │
│ Backup policy: [DailyBackup ▼]                                   │
│                                                                       │
│ Protocol:                                                            │
│ ● SMB (Windows/Linux/macOS — most common)                        │
│ ○ NFS (Linux only — requires Premium tier + VNet)                │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING FILE SHARES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Click on file share → Browse files:                                │
│                                                                       │
│ [+ Add directory] [Upload] [Connect] [Snapshot]                   │
│                                                                       │
│ ├── documents/                                                    │
│ │   ├── report-2024.pdf (512 KB)                                │
│ │   └── budget.xlsx (1.2 MB)                                    │
│ ├── config/                                                       │
│ │   └── app-settings.json (2 KB)                                │
│ └── readme.txt (1 KB)                                             │
│                                                                       │
│ Properties:                                                          │
│ ├── Quota: 5120 GiB (can increase up to 100 TiB)               │
│ ├── Used: 1.8 GiB                                                │
│ ├── Protocol: SMB                                                 │
│ └── Tier: Transaction optimized                                   │
│                                                                       │
│ ⚡ Quota limits the max size. Increase anytime in Properties.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Connecting to File Shares

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECTING (Mount File Share)                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal → File share → Connect → Shows mount commands             │
│                                                                       │
│ ── Windows ──                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ # PowerShell (run as admin):                                │  │
│ │                                                              │  │
│ │ # Test connectivity (port 445)                              │  │
│ │ Test-NetConnection -ComputerName stmyappprod.file.core.     │  │
│ │   windows.net -Port 445                                     │  │
│ │                                                              │  │
│ │ # Mount as Z: drive                                         │  │
│ │ net use Z: \\stmyappprod.file.core.windows.net\shared-files│  │
│ │   /user:stmyappprod <storage-account-key>                  │  │
│ │                                                              │  │
│ │ # Or use cmdkey for persistent mount:                       │  │
│ │ cmdkey /add:stmyappprod.file.core.windows.net               │  │
│ │   /user:stmyappprod /pass:<storage-account-key>            │  │
│ │ New-PSDrive -Name Z -PSProvider FileSystem                  │  │
│ │   -Root "\\stmyappprod.file.core.windows.net\shared-files" │  │
│ │   -Persist                                                   │  │
│ │                                                              │  │
│ │ ⚡ Port 445 must be open (some ISPs block it!)              │  │
│ │   Use VPN or VNet if port 445 is blocked.                  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Linux ──                                                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ # Install cifs-utils                                        │  │
│ │ sudo apt install cifs-utils                                  │  │
│ │                                                              │  │
│ │ # Create mount point                                        │  │
│ │ sudo mkdir /mnt/azure-share                                  │  │
│ │                                                              │  │
│ │ # Mount                                                      │  │
│ │ sudo mount -t cifs //stmyappprod.file.core.windows.net/     │  │
│ │   shared-files /mnt/azure-share -o                          │  │
│ │   vers=3.0,username=stmyappprod,password=<key>,             │  │
│ │   serverino,nosharesock,actimeo=30                          │  │
│ │                                                              │  │
│ │ # Add to /etc/fstab for persistent mount                   │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── macOS ──                                                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Finder → Go → Connect to Server                           │  │
│ │ smb://stmyappprod.file.core.windows.net/shared-files       │  │
│ │ Username: stmyappprod                                        │  │
│ │ Password: <storage-account-key>                             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Azure File Sync

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE FILE SYNC                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is File Sync?                                                   │
│ ├── Sync on-premises file servers WITH Azure file shares         │
│ ├── Keep using your local file server (fast local access)       │
│ ├── Azure becomes the "master copy" in the cloud                │
│ ├── Cloud tiering: Rarely used files stored only in cloud       │
│ ├── Multi-site sync: Multiple offices sync through Azure        │
│ └── Disaster recovery: If local server dies, data is in Azure   │
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ Office A (file server)  ←→  Azure File Share  ←→  Office B │  │
│ │ ┌─────────────────┐      ┌──────────────┐  ┌──────────────┐│  │
│ │ │ D:\SharedData    │ sync │ shared-files  │  │ E:\SharedData││  │
│ │ │ ├── docs/       │ ←→  │ ├── docs/    │  │ ├── docs/    ││  │
│ │ │ ├── images/     │      │ ├── images/  │  │ ├── images/  ││  │
│ │ │ └── config/     │      │ └── config/  │  │ └── config/  ││  │
│ │ │                  │      │              │  │              ││  │
│ │ │ Agent installed  │      │ Cloud master │  │ Agent install ││  │
│ │ └─────────────────┘      └──────────────┘  └──────────────┘│  │
│ │                                                              │  │
│ │ Cloud tiering example:                                       │  │
│ │ ├── Local disk: 500 GB total                                │  │
│ │ ├── Frequently used files: stored locally (fast access)    │  │
│ │ ├── Rarely used files: only in Azure (stub on local disk)  │  │
│ │ └── When user opens stub → auto-downloads from Azure       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Setup:                                                               │
│ 1. Create Storage Sync Service (Portal → Storage Sync Service)   │
│ 2. Create Sync Group (links Azure file share to server endpoints)│
│ 3. Install Azure File Sync Agent on Windows Server               │
│ 4. Register server with Storage Sync Service                      │
│ 5. Add server endpoint (folder path on server)                    │
│ 6. Configure cloud tiering policies                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Snapshots & Backup

```
┌─────────────────────────────────────────────────────────────────────┐
│           SNAPSHOTS & BACKUP                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Snapshots:                                                           │
│ ├── Point-in-time, read-only copy of entire file share           │
│ ├── Portal → File share → Snapshots → Create snapshot          │
│ ├── Up to 200 snapshots per share                                │
│ ├── Restore individual files or entire share from snapshot      │
│ ├── Incremental (only changes stored — space efficient)        │
│ └── Great for quick recovery from accidental changes            │
│                                                                       │
│ Azure Backup:                                                        │
│ ├── Automated, policy-based backups                              │
│ ├── Portal → File share → Backup → Configure backup           │
│ ├── Backup vault: Select or create Recovery Services vault      │
│ ├── Policy: Daily/weekly/monthly retention                      │
│ ├── Restore: Full share or individual files                     │
│ ├── Soft delete: Deleted backups retained for 14 days          │
│ └── ⚡ Different from snapshots: managed, scheduled, retained  │
│                                                                       │
│ Soft Delete (file share):                                            │
│ ├── Portal → Storage Account → File shares → Soft delete      │
│ ├── Deleted file shares recoverable for 1-365 days             │
│ └── Default: 7 days retention                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_storage_share" "main" {
  name                 = "shared-files"
  storage_account_name = azurerm_storage_account.main.name
  quota                = 100  # GB
  access_tier          = "TransactionOptimized"
}

resource "azurerm_storage_share_directory" "documents" {
  name             = "documents"
  storage_share_id = azurerm_storage_share.main.id
}
```

### Bicep

```bicep
resource fileShare 'Microsoft.Storage/storageAccounts/fileServices/shares@2023-01-01' = {
  name: '${storageAccount.name}/default/shared-files'
  properties: {
    shareQuota: 100
    accessTier: 'TransactionOptimized'
  }
}
```

---

## Part 7: az CLI Reference

```bash
# Create file share
az storage share-rm create \
  --storage-account stmyappprod \
  --name shared-files \
  --quota 100 \
  --access-tier TransactionOptimized

# List file shares
az storage share-rm list --storage-account stmyappprod --output table

# Create directory
az storage directory create \
  --account-name stmyappprod \
  --share-name shared-files \
  --name documents

# Upload file
az storage file upload \
  --account-name stmyappprod \
  --share-name shared-files \
  --source ./report.pdf \
  --path documents/report.pdf

# Download file
az storage file download \
  --account-name stmyappprod \
  --share-name shared-files \
  --path documents/report.pdf \
  --dest ./report.pdf

# List files
az storage file list \
  --account-name stmyappprod \
  --share-name shared-files \
  --output table

# Create snapshot
az storage share snapshot --account-name stmyappprod --name shared-files

# Delete file share
az storage share-rm delete --storage-account stmyappprod --name shared-files --yes
```

---

## Part 8: Real-World Patterns

```
Pattern 1: Shared Config for VMs
├── Azure File Share: "config" (Transaction Optimized)
├── Mount on all VMs in VMSS via cloud-init
├── App reads config.json from /mnt/config/
├── Update config in one place → all VMs see changes
└── Access: Managed identity + RBAC

Pattern 2: Hybrid File Sync (Branch Offices)
├── Azure File Share: "company-data" (100 TB)
├── File Sync Agent on each office's Windows Server
├── Cloud tiering: 20% free space policy
├── Frequently used files cached locally (fast)
├── Old files stored only in Azure (auto-recalled on access)
└── Disaster recovery: If server dies, data is safe in Azure
```

---

## Quick Reference

```
Azure Files = Cloud file shares (SMB/NFS)
SMB = Windows + Linux + macOS (port 445)
NFS = Linux only (Premium tier + VNet required)

Tiers: Premium (SSD) → Transaction Optimized → Cool → Cold
Premium = Provisioned capacity (pay for allocated)
Standard = Pay per transaction + storage used

File Sync = Sync on-prem servers with Azure file shares
Cloud Tiering = Keep hot files local, cold files in Azure
Snapshots = Point-in-time copy of entire file share
Backup = Automated, policy-based (Recovery Services vault)

Max share size: 100 TiB
Max file size: 4 TiB (SMB) / 16 TiB (NFS)
```

---

## What's Next?

Next chapter: [Chapter 25: Managed Disks](25-managed-disks.md) — Understand Azure's block storage for VM disks, disk types, snapshots, and encryption.
