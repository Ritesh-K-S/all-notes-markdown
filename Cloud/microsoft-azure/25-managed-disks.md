# Chapter 25: Azure Managed Disks

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Managed Disks Fundamentals](#part-1-managed-disks-fundamentals)
- [Part 2: Disk Types & Performance](#part-2-disk-types--performance)
- [Part 3: Creating a Managed Disk (Portal Walkthrough)](#part-3-creating-a-managed-disk-portal-walkthrough)
- [Part 4: Attaching Disks to VMs](#part-4-attaching-disks-to-vms)
- [Part 5: Snapshots & Images](#part-5-snapshots--images)
- [Part 6: Disk Encryption](#part-6-disk-encryption)
- [Part 7: Shared Disks](#part-7-shared-disks)
- [Part 8: Terraform & Bicep](#part-8-terraform--bicep)
- [Part 9: az CLI Reference](#part-9-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Managed Disks are block-level storage volumes used as virtual hard disks for Azure VMs. Think of them as the hard drive of your cloud server. Azure manages the underlying storage infrastructure — you just pick the disk type, size, and attach it to your VM.

```
What you'll learn:
├── Managed Disks Fundamentals
│   ├── What are Managed Disks
│   ├── OS disk vs Data disk vs Temp disk
│   └── Managed vs Unmanaged (legacy)
├── Disk Types (Ultra, Premium SSD, Standard SSD, Standard HDD)
├── Creating & Attaching Disks (Portal)
├── Snapshots & Images
├── Disk Encryption (SSE, ADE, CMK)
├── Shared Disks (multi-VM attach)
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Managed Disks Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGED DISKS CONCEPT                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VM disk layout:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Virtual Machine: vm-web-prod-01                             │  │
│ │                                                              │  │
│ │ ┌─────────────────┐ OS Disk (required)                     │  │
│ │ │ /dev/sda (Linux) │ Contains: OS + boot files             │  │
│ │ │ C: (Windows)    │ Size: 30-128 GB (depends on image)    │  │
│ │ │ Premium SSD P10  │ ⚡ Always Managed Disk                │  │
│ │ └─────────────────┘                                         │  │
│ │                                                              │  │
│ │ ┌─────────────────┐ Temp Disk (free, ephemeral)            │  │
│ │ │ /dev/sdb (Linux) │ Data LOST on VM stop/resize/move!     │  │
│ │ │ D: (Windows)    │ For: swap, temp files, page files     │  │
│ │ │ Local SSD/HDD   │ Size: Varies by VM size                │  │
│ │ └─────────────────┘ ⚠️ NEVER store important data here!   │  │
│ │                                                              │  │
│ │ ┌─────────────────┐ Data Disk 1 (optional)                 │  │
│ │ │ /dev/sdc (Linux) │ Additional storage for your data      │  │
│ │ │ E: (Windows)    │ App data, databases, logs              │  │
│ │ │ Premium SSD P30  │ Survives VM stop/restart              │  │
│ │ └─────────────────┘                                         │  │
│ │                                                              │  │
│ │ ┌─────────────────┐ Data Disk 2 (optional)                 │  │
│ │ │ /dev/sdd (Linux) │ Can add multiple data disks           │  │
│ │ │ F: (Windows)    │ Max depends on VM size                 │  │
│ │ │ Standard SSD E30│                                         │  │
│ │ └─────────────────┘                                         │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Managed vs Unmanaged disks:                                          │
│ ├── Managed: Azure handles storage account (you just pick type) │
│ ├── Unmanaged: You manage storage account + VHD files (LEGACY) │
│ └── ⚡ Always use Managed Disks! Unmanaged is deprecated.       │
│                                                                       │
│ ⚡ AWS equivalent: EBS (Elastic Block Store)                        │
│ ⚡ GCP equivalent: Persistent Disk                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Disk Types & Performance

```
┌─────────────────────────────────────────────────────────────────────┐
│           DISK TYPES                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┬────────┬──────────┬──────────┬───────────────┐   │
│ │ Type         │ Media  │ Max IOPS │ Max MB/s │ Use case       │   │
│ ├──────────────┼────────┼──────────┼──────────┼───────────────┤   │
│ │ Ultra Disk   │ SSD    │ 160,000  │ 4,000    │ SAP, databases │   │
│ │ Premium SSD  │ SSD    │ 20,000   │ 900      │ Production VMs │   │
│ │ Premium SSD  │ SSD    │ 80,000   │ 1,200    │ Perf-sensitive │   │
│ │ v2           │        │          │          │ workloads      │   │
│ │ Standard SSD │ SSD    │ 6,000    │ 750      │ Dev/test, web  │   │
│ │ Standard HDD │ HDD    │ 2,000    │ 500      │ Backup, archive│   │
│ └──────────────┴────────┴──────────┴──────────┴───────────────┘   │
│                                                                       │
│ Premium SSD sizes & pricing (common):                                │
│ ┌───────┬──────────┬──────┬────────┬──────────────────────────┐   │
│ │ SKU   │ Size     │ IOPS │ MB/s   │ Monthly cost (approx)   │   │
│ ├───────┼──────────┼──────┼────────┼──────────────────────────┤   │
│ │ P4    │ 32 GB    │ 120  │ 25     │ ~$6                      │   │
│ │ P6    │ 64 GB    │ 240  │ 50     │ ~$10                     │   │
│ │ P10   │ 128 GB   │ 500  │ 100    │ ~$18                     │   │
│ │ P15   │ 256 GB   │ 1,100│ 125    │ ~$34                     │   │
│ │ P20   │ 512 GB   │ 2,300│ 150    │ ~$65                     │   │
│ │ P30   │ 1 TB     │ 5,000│ 200    │ ~$122                    │   │
│ │ P40   │ 2 TB     │ 7,500│ 250    │ ~$236                    │   │
│ │ P50   │ 4 TB     │10,000│ 250    │ ~$458                    │   │
│ └───────┴──────────┴──────┴────────┴──────────────────────────┘   │
│                                                                       │
│ Decision guide:                                                      │
│ ├── Dev/test → Standard SSD (good enough, cheapest SSD)         │
│ ├── Production web/API → Premium SSD P10-P30                    │
│ ├── Database (SQL Server, PostgreSQL) → Premium SSD P30+        │
│ ├── Highest performance → Ultra Disk or Premium SSD v2          │
│ ├── Backup/archive → Standard HDD                               │
│ └── ⚡ IOPS and throughput scale with disk size!                 │
│     Bigger disk = more IOPS = faster.                            │
│                                                                       │
│ Bursting:                                                            │
│ ├── Small Premium SSDs (P1-P20) can burst above baseline IOPS  │
│ ├── Burst up to 3,500 IOPS / 170 MB/s for short periods       │
│ ├── Great for boot/startup and occasional spikes                │
│ └── Credit-based (build up credits during idle time)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating a Managed Disk (Portal Walkthrough)

```
Console → Disks → Create

┌─────────────────────────────────────────────────────────────────────┐
│           BASICS                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-compute-prod ▼]                                │
│                                                                       │
│ Disk name: [disk-data-web-01]                                      │
│ Region: [Central India ▼]                                          │
│ Availability zone: [1 ▼] (must match VM's zone!)                 │
│                                                                       │
│ Source type:                                                         │
│ ● None (empty disk)                                                │
│ ○ Snapshot (create from existing snapshot)                        │
│ ○ Storage blob (from VHD in blob storage)                        │
│ ○ Upload (upload a VHD from local)                               │
│                                                                       │
│ Size: [Premium SSD P30 - 1024 GiB, 5000 IOPS, 200 MB/s ▼]     │
│ [Change size] → Select disk type and size                        │
│                                                                       │
│ Performance plus (higher tier): ☐                                 │
│ ⚡ Temporarily boost IOPS/throughput to a higher tier.            │
│                                                                       │
│ ── Encryption ──                                                    │
│ Encryption type: [Encryption at-rest with platform-managed key ▼]│
│ ○ Platform-managed key (Microsoft manages — default, easiest)  │
│ ○ Customer-managed key (your key from Key Vault)               │
│ ○ Platform + customer-managed key (double encryption)          │
│                                                                       │
│ ── Networking ──                                                    │
│ ○ Public endpoint (all networks)                                  │
│ ○ Public endpoint (selected networks)                             │
│ ● Disable public access, enable private access                   │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Attaching Disks to VMs

```
┌─────────────────────────────────────────────────────────────────────┐
│           ATTACH / DETACH DISKS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal → VM → Settings → Disks → [Attach existing disk]        │
│                                                                       │
│ Or during VM creation → Disks tab → [Create and attach new]    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Data disks:                                                  │  │
│ │                                                              │  │
│ │ LUN │ Name              │ Size    │ Type         │ Caching  │  │
│ │ 0   │ disk-data-web-01  │ 1024 GB │ Premium SSD  │ ReadOnly │  │
│ │ 1   │ disk-logs-web-01  │ 256 GB  │ Standard SSD │ None     │  │
│ │                                                              │  │
│ │ LUN = Logical Unit Number (0-based)                         │  │
│ │ Shows as /dev/sdc, /dev/sdd (Linux) or E:, F: (Windows)   │  │
│ │                                                              │  │
│ │ Host caching:                                                │  │
│ │ ├── None: No caching (write-heavy workloads)               │  │
│ │ ├── ReadOnly: Cache reads (most common for data disks) ✅  │  │
│ │ └── Read/Write: Cache both (OS disk only!)                 │  │
│ │                                                              │  │
│ │ ⚡ For databases: Use ReadOnly or None caching.             │  │
│ │   For OS disk: Read/Write is fine.                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ After attaching (Linux):                                             │
│ 1. Find disk: lsblk (shows new disk, e.g., /dev/sdc)            │
│ 2. Partition: sudo fdisk /dev/sdc → n → p → w                 │
│ 3. Format: sudo mkfs.ext4 /dev/sdc1                              │
│ 4. Mount: sudo mount /dev/sdc1 /mnt/data                        │
│ 5. Persist: Add to /etc/fstab (use UUID, not /dev/sdX)          │
│                                                                       │
│ Detach: Portal → VM → Disks → [X] next to disk → Save         │
│ ⚡ Detach before deleting a disk! Data on disk is preserved.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Snapshots & Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           SNAPSHOTS & IMAGES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Snapshot:                                                            │
│ ├── Point-in-time copy of a Managed Disk                         │
│ ├── Read-only (cannot modify a snapshot)                         │
│ ├── Create new disk from snapshot (restore/clone)               │
│ ├── Copy to another region for DR                                │
│ ├── Incremental snapshots (only changes — saves space!)        │
│ └── Portal → Disk → Create snapshot                             │
│                                                                       │
│ Create snapshot:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Name: snap-disk-data-web-01-2024-01-15                      │  │
│ │ Snapshot type:                                               │  │
│ │   ● Incremental (cheaper — stores only changes) ✅         │  │
│ │   ○ Full (complete copy — more expensive)                  │  │
│ │ Source disk: disk-data-web-01                                │  │
│ │ [Create]                                                     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Image (for VM templates):                                            │
│ ├── Generalized image = Template for creating new VMs            │
│ ├── Contains OS disk + optional data disks                      │
│ ├── Use Azure Compute Gallery for sharing images                │
│ │   (across subscriptions, tenants, regions)                   │
│ └── Steps: Prepare VM → Generalize → Capture → Use in VMSS   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Disk Encryption

```
┌─────────────────────────────────────────────────────────────────────┐
│           DISK ENCRYPTION                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Server-Side Encryption (SSE) — enabled by default!             │
│    ├── All managed disks encrypted at rest (AES-256)              │
│    ├── Transparent (no performance impact)                        │
│    ├── Microsoft-managed keys (default) or CMK (Key Vault)      │
│    └── You don't need to do anything — it's automatic!          │
│                                                                       │
│ 2. Azure Disk Encryption (ADE) — additional layer                 │
│    ├── Encrypts inside the VM using BitLocker (Windows)          │
│    │   or dm-crypt (Linux)                                       │
│    ├── Keys stored in Key Vault                                   │
│    ├── Adds CPU overhead (2-3%)                                   │
│    └── Required for some compliance standards                    │
│                                                                       │
│ 3. Encryption at host                                                │
│    ├── Encrypts temp disk and disk caches                        │
│    ├── Data encrypted before reaching storage                    │
│    └── End-to-end encryption                                     │
│                                                                       │
│ Summary:                                                             │
│ ├── SSE: Default, always on, transparent ← most use this       │
│ ├── ADE: Extra layer for compliance (BitLocker/dm-crypt)       │
│ └── At host: Encrypts temp disk + caches too                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Shared Disks

```
┌─────────────────────────────────────────────────────────────────────┐
│           SHARED DISKS                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Attach ONE disk to MULTIPLE VMs simultaneously!                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Shared Disk: disk-shared-cluster                             │  │
│ │ Type: Premium SSD P30 (1 TB)                                │  │
│ │ Max shares: 3                                                │  │
│ │                                                              │  │
│ │ ├── VM 1: vm-cluster-01 ──┐                                │  │
│ │ ├── VM 2: vm-cluster-02 ──┤── All access same disk         │  │
│ │ └── VM 3: vm-cluster-03 ──┘                                │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Use cases:                                                           │
│ ├── Clustered databases (SQL Server Failover Cluster)            │
│ ├── Windows Server Failover Clustering (WSFC)                    │
│ └── Clustered applications that manage their own I/O             │
│                                                                       │
│ ⚡ Requires cluster software (not just "shared drive")!           │
│   Applications must handle concurrent access coordination.       │
│                                                                       │
│ Enable: Disk → Properties → Max shares: [2-10]                  │
│ Supported on: Premium SSD and Ultra Disk only                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_managed_disk" "data" {
  name                 = "disk-data-web-01"
  location             = azurerm_resource_group.main.location
  resource_group_name  = azurerm_resource_group.main.name
  storage_account_type = "Premium_LRS"
  disk_size_gb         = 256
  create_option        = "Empty"
  zone                 = "1"
}

resource "azurerm_virtual_machine_data_disk_attachment" "data" {
  managed_disk_id    = azurerm_managed_disk.data.id
  virtual_machine_id = azurerm_linux_virtual_machine.main.id
  lun                = 0
  caching            = "ReadOnly"
}

# Snapshot
resource "azurerm_snapshot" "data" {
  name                = "snap-disk-data-web-01"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  create_option       = "Copy"
  source_uri          = azurerm_managed_disk.data.id
}
```

### Bicep

```bicep
// Managed Disk
resource dataDisk 'Microsoft.Compute/disks@2023-10-02' = {
  name: 'disk-data-web-01'
  location: resourceGroup().location
  sku: {
    name: 'Premium_LRS'
  }
  zones: ['1']
  properties: {
    diskSizeGB: 256
    creationData: {
      createOption: 'Empty'
    }
  }
}

// Snapshot
resource snapshot 'Microsoft.Compute/snapshots@2023-10-02' = {
  name: 'snap-disk-data-web-01'
  location: resourceGroup().location
  properties: {
    creationData: {
      createOption: 'Copy'
      sourceResourceId: dataDisk.id
    }
  }
}
```
az disk create \
  --name disk-data-web-01 \
  --resource-group rg-compute-prod \
  --location centralindia \
  --sku Premium_LRS \
  --size-gb 256 \
  --zone 1

# Attach disk to VM
az vm disk attach \
  --vm-name vm-web-prod-01 \
  --resource-group rg-compute-prod \
  --name disk-data-web-01 \
  --lun 0 \
  --caching ReadOnly

# Detach disk from VM
az vm disk detach \
  --vm-name vm-web-prod-01 \
  --resource-group rg-compute-prod \
  --name disk-data-web-01

# Create snapshot
az snapshot create \
  --name snap-disk-data-web-01 \
  --resource-group rg-compute-prod \
  --source disk-data-web-01 \
  --incremental

# Create disk from snapshot
az disk create \
  --name disk-restored-web-01 \
  --resource-group rg-compute-prod \
  --source snap-disk-data-web-01

# Resize disk (can only increase)
az disk update \
  --name disk-data-web-01 \
  --resource-group rg-compute-prod \
  --size-gb 512

# Delete disk
az disk delete \
  --name disk-data-web-01 \
  --resource-group rg-compute-prod \
  --yes

# Delete snapshot
az snapshot delete \
  --name snap-disk-data-web-01 \
  --resource-group rg-compute-prod
```

---

## Part 10: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: HIGH-PERFORMANCE DATABASE VM                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VM: Standard_E16s_v5 (16 vCores, 128 GB RAM)                     │
│ ├── OS Disk: Premium SSD P10 (128 GB) — ReadWrite caching       │
│ ├── Data Disk 1: Premium SSD P30 (1 TB) — ReadOnly caching     │
│ ├── Data Disk 2: Premium SSD P30 (1 TB) — ReadOnly caching     │
│ ├── Log Disk: Premium SSD P20 (512 GB) — None caching          │
│ └── Temp Disk: Used for tempdb (free, fast)                     │
│                                                                       │
│ ⚡ Data disks: ReadOnly cache (reads from local SSD)             │
│ ⚡ Log disk: No cache (write-intensive, sequential)              │
│ ⚡ Stripe data disks for higher IOPS (Storage Spaces / LVM)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN: SNAPSHOT-BASED DR                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Daily: Snapshot all production VM disks                             │
│ Retention: Keep 7 daily, 4 weekly, 12 monthly                     │
│                                                                       │
│ On disaster:                                                        │
│ 1. Create new disks from latest snapshots                         │
│ 2. Create new VM with these disks                                 │
│ 3. Start VM → Production restored in minutes                    │
│                                                                       │
│ Automate with Azure Backup or Azure Policy!                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Managed Disk = Block storage volume for VMs
OS Disk = Boot disk (required, 1 per VM)
Data Disk = Additional storage (optional, multiple per VM)
Temp Disk = Ephemeral, free, data lost on VM operations

Types: Ultra > Premium SSD v2 > Premium SSD > Standard SSD > Standard HDD
IOPS/throughput = scales with disk SIZE (bigger = faster)
Bursting = Small disks can burst above baseline temporarily

Snapshot = Point-in-time copy (incremental = cheaper)
Shared Disk = Attach 1 disk to multiple VMs (clusters only)

Encryption:
  SSE = Always on, transparent (default)
  ADE = BitLocker/dm-crypt (compliance)
  At host = Encrypts temp disk + caches

Caching: None (write-heavy) | ReadOnly (data) | Read/Write (OS only)
```

---

## What's Next?

Next chapter: [Chapter 26: Azure Data Lake Storage Gen2](26-data-lake-storage.md) — Hierarchical namespace, ACLs, and analytics-optimized storage.
