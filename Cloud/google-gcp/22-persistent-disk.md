# Chapter 22: Persistent Disk & Local SSD

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Persistent Disk Fundamentals](#part-1-persistent-disk-fundamentals)
- [Part 2: Disk Types — Deep Dive](#part-2-disk-types--deep-dive)
- [Part 3: Creating & Attaching Disks](#part-3-creating--attaching-disks)
- [Part 4: Disk Resizing](#part-4-disk-resizing)
- [Part 5: Snapshots](#part-5-snapshots)
- [Part 6: Machine Images](#part-6-machine-images)
- [Part 7: Custom Images](#part-7-custom-images)
- [Part 8: Local SSD](#part-8-local-ssd)
- [Part 9: Hyperdisk](#part-9-hyperdisk)
- [Part 10: Disk Encryption](#part-10-disk-encryption)
- [Part 11: Performance Tuning](#part-11-performance-tuning)
- [Part 12: Terraform & CLI](#part-12-terraform--cli)
- [Part 13: Real-World Patterns](#part-13-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Persistent Disk is GCP's block storage service — network-attached durable disks for Compute Engine VMs. Unlike local storage, Persistent Disks survive VM deletion and can be detached/reattached. This chapter covers all disk types, snapshots, images, Local SSD, and the newer Hyperdisk family.

```
What you'll learn:
├── Persistent Disk Fundamentals
│   ├── Network-attached vs local storage
│   ├── Zonal vs Regional disks
│   ├── Read-only multi-attach
│   └── Boot disk vs data disk
├── Disk Types
│   ├── pd-standard (HDD) — cheapest
│   ├── pd-balanced (SSD) — best value ✅
│   ├── pd-ssd (SSD) — high IOPS
│   └── pd-extreme (SSD) — highest IOPS
├── Snapshots
│   ├── Incremental snapshots
│   ├── Snapshot schedules
│   ├── Cross-region snapshots
│   └── Restore from snapshot
├── Images
│   ├── Machine images (full VM capture)
│   ├── Custom images (OS disk only)
│   └── Image families
├── Local SSD
│   ├── Physically attached NVMe
│   ├── Extreme performance, ephemeral
│   └── Data LOST on VM stop/delete
├── Hyperdisk
│   ├── Hyperdisk Extreme, Throughput, Balanced
│   ├── Dynamic provisioning
│   └── Independently scalable IOPS & throughput
├── Encryption (Google-managed, CMEK, CSEK)
├── Performance tuning
├── Terraform & CLI
└── Real-world patterns
```

---

## Part 1: Persistent Disk Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS PERSISTENT DISK?                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Persistent Disk = network-attached block storage for VMs.          │
│ Think of it as a virtual hard drive, but stored on Google's       │
│ storage infrastructure, NOT physically inside the VM host.        │
│                                                                       │
│ ┌─────────────┐          ┌───────────────────────────┐            │
│ │  VM Instance │◄────────►│  Persistent Disk           │            │
│ │  (Compute)   │ network  │  (Storage infrastructure)  │            │
│ └─────────────┘          └───────────────────────────┘            │
│                                                                       │
│ Key properties:                                                     │
│ ├── DURABLE: Data persists even if VM is deleted                 │
│ │   (unless "Delete boot disk" checkbox is checked)              │
│ ├── NETWORK-ATTACHED: Connected via Google's network             │
│ │   (not physically inside the host machine)                     │
│ ├── RESIZABLE: Can increase size without downtime                │
│ │   ⚠️ Cannot DECREASE size — ever!                               │
│ ├── SNAPSHOTS: Point-in-time backups, incremental               │
│ ├── ENCRYPTED: Always encrypted at rest (Google-managed default) │
│ └── INDEPENDENT lifecycle from VM (can detach and reattach)     │
│                                                                       │
│ AWS equivalent: EBS (Elastic Block Store)                          │
│ Azure equivalent: Managed Disks                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Zonal vs Regional Persistent Disk

```
┌─────────────────────────────────────────────────────────────────────┐
│           ZONAL vs REGIONAL                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ZONAL PERSISTENT DISK (default):                                    │
│ ├── Data stored in a single zone                                 │
│ ├── VM must be in the SAME zone as the disk                     │
│ ├── If the zone goes down → disk is unavailable                 │
│ ├── Lower cost                                                   │
│ └── Sufficient for most workloads                                │
│                                                                       │
│   ┌────────────┐                                                   │
│   │  Zone A     │                                                   │
│   │ ┌────┐┌──┐ │                                                   │
│   │ │ VM ││PD│ │  ← Both in same zone                             │
│   │ └────┘└──┘ │                                                   │
│   └────────────┘                                                   │
│                                                                       │
│ REGIONAL PERSISTENT DISK:                                           │
│ ├── Data replicated across TWO zones in the same region         │
│ ├── Synchronous replication (RPO = 0)                            │
│ ├── Survives single zone failure                                 │
│ ├── Used with regional MIGs for HA                              │
│ ├── 2x the cost of zonal (paying for replication)              │
│ ├── Slightly higher write latency (sync replication)            │
│ └── Can force-attach to VM in the other zone during failover   │
│                                                                       │
│   ┌────────────┐    ┌────────────┐                                │
│   │  Zone A     │    │  Zone B     │                                │
│   │ ┌────┐┌──┐ │    │     ┌──┐   │                                │
│   │ │ VM ││PD│◄├────┼─────►PD│   │  ← Sync replication            │
│   │ └────┘└──┘ │    │     └──┘   │                                │
│   └────────────┘    └────────────┘                                │
│                                                                       │
│ When to use Regional:                                               │
│ ├── Databases that need zone-level HA                            │
│ ├── Stateful MIGs (regional instance groups)                    │
│ ├── Critical data that cannot tolerate zone failure             │
│ └── Compliance requiring multi-zone redundancy                  │
│                                                                       │
│ AWS equivalent: EBS has NO direct equivalent (single-AZ only).   │
│   AWS uses EBS Snapshots + cross-AZ restore for HA.             │
│ Azure equivalent: Zone-redundant disks (ZRS Managed Disks)       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Boot Disk vs Data Disk

```
┌─────────────────────────────────────────────────────────────────────┐
│           BOOT DISK vs DATA DISK                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ BOOT DISK:                                                          │
│ ├── Contains the OS (Linux/Windows)                              │
│ ├── Created automatically when you create a VM                  │
│ ├── Default: deleted when VM is deleted                          │
│ │   ⚡ Uncheck "Delete boot disk when VM is deleted" to keep it │
│ ├── Can be replaced (stop VM → detach → attach new boot disk)  │
│ ├── Default size: 10 GB (Linux), 50 GB (Windows)               │
│ └── Max: same as disk type limit (up to 64 TB)                 │
│                                                                       │
│ DATA DISK (additional disk):                                       │
│ ├── Extra disk(s) attached to VM for application data           │
│ ├── NOT deleted when VM is deleted (by default)                 │
│ ├── Can be detached and reattached to another VM               │
│ ├── Must be formatted and mounted by the OS                    │
│ ├── A VM can have up to 128 disks attached                     │
│ └── Each can be a different disk type                           │
│                                                                       │
│ Multi-attach (read-only):                                          │
│ ├── A single PD can be attached to MULTIPLE VMs in read-only   │
│ ├── Use case: shared config files, ML model weights             │
│ ├── Max 10 VMs can attach the same read-only disk              │
│ └── ⚠️ Read-write is ONE VM only (no shared-write)              │
│                                                                       │
│ AWS equivalent: EBS multi-attach (io1/io2, read-WRITE to 16)   │
│ ⚡ GCP read-only only. AWS supports read-write multi-attach     │
│   but only on io1/io2 with cluster-aware file systems.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Disk Types — Deep Dive

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              PERSISTENT DISK TYPES                                              │
├────────────────┬─────────────┬─────────────┬─────────────┬──────────────────────────────────┤
│ Property       │ pd-standard │ pd-balanced │ pd-ssd      │ pd-extreme                       │
│                │ (HDD)       │ (SSD)       │ (SSD)       │ (SSD)                            │
├────────────────┼─────────────┼─────────────┼─────────────┼──────────────────────────────────┤
│ Backing        │ Standard    │ SSD         │ SSD         │ SSD                              │
│ storage        │ HDD         │             │             │                                  │
│                │             │             │             │                                  │
│ Min size       │ 10 GB       │ 10 GB       │ 10 GB       │ 500 GB                           │
│ Max size       │ 64 TB       │ 64 TB       │ 64 TB       │ 64 TB                            │
│                │             │             │             │                                  │
│ Max IOPS       │ 7,500 R     │ 80,000 R    │ 100,000 R   │ 120,000 R                        │
│ (per disk)     │ 15,000 W    │ 30,000 W    │ 30,000 W    │ 120,000 W                        │
│                │             │             │             │                                  │
│ Max throughput │ 1,200 R     │ 1,200 R     │ 1,200 R     │ 2,200 R                          │
│ MB/s           │ 400 W       │ 1,200 W     │ 1,200 W     │ 2,200 W                          │
│                │             │             │             │                                  │
│ IOPS per GB    │ 0.75 R      │ 6 R         │ 30 R        │ Provisioned                      │
│                │ 1.5 W       │ 6 W         │ 30 W        │ (you choose)                     │
│                │             │             │             │                                  │
│ Latency        │ ~ms         │ Sub-ms      │ Sub-ms      │ Sub-ms                           │
│                │             │             │             │                                  │
│ Cost (per GB   │ $0.040      │ $0.100      │ $0.170      │ $0.125/GB +                      │
│ per month)     │ (cheapest)  │ (best ✅)   │             │ $0.01/IOPS                       │
│                │             │             │             │                                  │
│ Regional avail │ ✅ Yes      │ ✅ Yes      │ ✅ Yes      │ ❌ Zonal only                     │
│                │             │             │             │                                  │
│ Use case       │ Logs, cold  │ General     │ Databases,  │ SAP HANA,                        │
│                │ data, bulk  │ purpose,    │ high-perf   │ Oracle DB,                       │
│                │ storage     │ boot disks  │ apps        │ extreme IOPS                     │
├────────────────┴─────────────┴─────────────┴─────────────┴──────────────────────────────────┤
│                                                                                               │
│ ⚡ IOPS SCALE WITH DISK SIZE (except pd-extreme):                                             │
│ A 100 GB pd-balanced gets: 100 × 6 = 600 IOPS                                               │
│ A 1 TB pd-balanced gets:  1000 × 6 = 6,000 IOPS                                             │
│ A 10 TB pd-balanced gets: 10000 × 6 = 60,000 IOPS (capped at 80,000)                       │
│                                                                                               │
│ ⚡ pd-extreme: YOU provision the IOPS independently of size.                                  │
│ A 500 GB pd-extreme with 50,000 IOPS — you pay for both.                                    │
│ Requires N2 or later machine types.                                                          │
│                                                                                               │
│ ⚡ VM machine type also limits IOPS! A small VM (e2-micro) cannot use                         │
│ all IOPS from a large pd-ssd. Check per-VM IOPS limits in docs.                             │
│                                                                                               │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Disk Type Comparison

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ GCP                    │ AWS EBS                │ Azure Managed Disk          │
├────────────────────────┼────────────────────────┼─────────────────────────────┤
│ pd-standard (HDD)      │ st1 (Throughput HDD)   │ Standard HDD (S/E Series)   │
│                        │ sc1 (Cold HDD)         │                             │
│ pd-balanced (SSD)      │ gp3 (General Purpose)  │ Standard SSD (E Series)     │
│ pd-ssd (SSD)           │ io1 (Provisioned IOPS) │ Premium SSD (P Series)      │
│ pd-extreme (SSD)       │ io2 Block Express      │ Ultra Disk                  │
│ Local SSD              │ Instance Store         │ Local NVMe/Temp Disk        │
│ Hyperdisk Extreme      │ io2 Block Express      │ Ultra Disk                  │
│ Hyperdisk Balanced     │ gp3                    │ Premium SSD v2              │
│ Hyperdisk Throughput   │ st1                    │ —                           │
└────────────────────────┴────────────────────────┴─────────────────────────────┘
```

---

## Part 3: Creating & Attaching Disks

### Console Walkthrough — Create Data Disk

```
Console → Compute Engine → Disks → Create Disk

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE A DISK                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [webapp-data-disk]                                            │
│ Description: [Data disk for web app database]                      │
│                                                                       │
│ Location:                                                           │
│   ○ Single zone (zonal)   ○ Regional (replicated)                  │
│   Region: [asia-south1]                                            │
│   Zone: [asia-south1-a]   (for zonal)                              │
│   Replica zone: [asia-south1-b]  (for regional)                   │
│                                                                       │
│ Disk source type:                                                   │
│   ○ Blank disk                                                      │
│   ○ Snapshot    → [select snapshot]                                │
│   ○ Image       → [select image]                                   │
│   ○ Disk (clone) → [select existing disk]                          │
│   ⚡ "Blank disk" = empty, needs formatting after attach           │
│   ⚡ "Snapshot" = restore from backup                               │
│   ⚡ "Image" = boot disk from OS image                              │
│                                                                       │
│ Disk type:                                                          │
│   ○ Balanced persistent disk (pd-balanced) ← recommended          │
│   ○ SSD persistent disk (pd-ssd)                                   │
│   ○ Standard persistent disk (pd-standard)                         │
│   ○ Extreme persistent disk (pd-extreme)                           │
│                                                                       │
│ Size (GB): [100]                                                    │
│   ⚡ Larger = more IOPS and throughput (scales linearly to cap)    │
│   ⚡ Cannot decrease later! Start smaller, grow as needed.         │
│                                                                       │
│ (pd-extreme only)                                                   │
│ Provisioned IOPS: [50000]                                          │
│                                                                       │
│ Encryption:                                                         │
│   ○ Google-managed encryption key (default)                        │
│   ○ Customer-managed encryption key (CMEK)                         │
│   ○ Customer-supplied encryption key (CSEK)                        │
│                                                                       │
│ Snapshot schedule: [none] or [select existing schedule]            │
│                                                                       │
│ Labels:                                                              │
│   environment: prod                                                │
│   team: backend                                                     │
│                                                                       │
│ [CREATE]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Attaching a Disk to a VM

```
┌─────────────────────────────────────────────────────────────────────┐
│           ATTACH DISK TO VM                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console: VM Instance → Edit → Additional disks → Add existing disk│
│                                                                       │
│ Mode:                                                                │
│   ○ Read/Write (default — one VM only)                             │
│   ○ Read-only (can attach to multiple VMs)                         │
│                                                                       │
│ Deletion rule:                                                      │
│   ○ Keep disk (default for data disks)                             │
│   ○ Delete disk when VM is deleted                                 │
│                                                                       │
│ After attaching — format and mount (Linux):                        │
│ ───────────────────────────────────────────                         │
│ # 1. List available disks                                          │
│ lsblk                                                               │
│ # Output: sdb (the new disk, no partitions yet)                   │
│                                                                       │
│ # 2. Format the disk (ONLY first time — destroys data!)           │
│ sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0 \  │
│   /dev/sdb                                                         │
│                                                                       │
│ # 3. Create mount point                                            │
│ sudo mkdir -p /mnt/data                                            │
│                                                                       │
│ # 4. Mount                                                         │
│ sudo mount -o discard,defaults /dev/sdb /mnt/data                 │
│                                                                       │
│ # 5. Set permissions                                               │
│ sudo chmod a+w /mnt/data                                           │
│                                                                       │
│ # 6. Add to /etc/fstab for auto-mount on reboot                  │
│ # Get UUID:                                                        │
│ sudo blkid /dev/sdb                                                │
│ # Add to fstab:                                                    │
│ echo "UUID=DISK_UUID /mnt/data ext4 discard,defaults,nofail 0 2" │
│   | sudo tee -a /etc/fstab                                        │
│                                                                       │
│ ⚡ Always use UUID in fstab (not /dev/sdb) because device names  │
│   can change on reboot!                                            │
│ ⚡ Use "nofail" option so VM still boots if disk is missing.       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Disk Resizing

```
┌─────────────────────────────────────────────────────────────────────┐
│           RESIZING PERSISTENT DISKS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ✅ Can INCREASE disk size (no downtime for most cases)              │
│ ❌ Can NEVER decrease disk size                                     │
│ ❌ Cannot change disk type (pd-standard → pd-ssd)                  │
│    ⚡ To change type: Snapshot → create new disk of desired type   │
│                                                                       │
│ Steps to resize:                                                    │
│                                                                       │
│ 1. Resize the disk (GCP side):                                    │
│    Console → Disks → [disk] → Edit → change size → Save          │
│    ⚡ No VM restart needed! Can resize while VM is running.        │
│                                                                       │
│ 2. Resize the filesystem (OS side — inside the VM):               │
│    # For ext4:                                                     │
│    sudo resize2fs /dev/sdb                                         │
│                                                                       │
│    # For XFS:                                                      │
│    sudo xfs_growfs /mnt/data                                      │
│                                                                       │
│    ⚡ The OS needs to know about the new space.                     │
│       GCP resizes the block device, but the filesystem             │
│       inside still sees the old size until you run resize.        │
│                                                                       │
│ 3. Verify:                                                         │
│    df -h /mnt/data                                                 │
│                                                                       │
│ Boot disk resize:                                                   │
│ ├── Same process, but resizes automatically on next reboot        │
│ │   for most OS images (auto-resize is configured)                │
│ └── Can also resize without reboot using resize2fs                │
│                                                                       │
│ AWS equivalent: EBS Modify Volume (also online, also can't shrink)│
│ Azure equivalent: Expand Managed Disk (VM must be deallocated     │
│   for OS disk, data disk can be online for some types)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Snapshots

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERSISTENT DISK SNAPSHOTS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Point-in-time backup of a Persistent Disk.                   │
│       Stored in Cloud Storage (managed by Google — you don't see  │
│       the bucket). Globally accessible — can restore in ANY       │
│       region.                                                      │
│                                                                       │
│ Key properties:                                                     │
│ ├── INCREMENTAL: Only changes since last snapshot are stored     │
│ │   First snapshot = full copy                                   │
│ │   Subsequent = only changed blocks (fast, cheap)               │
│ │   ⚡ Deleting an old snapshot moves its unique data to the      │
│ │     next snapshot — no data loss!                               │
│ │                                                                 │
│ ├── GLOBAL: Snapshot is a global resource                        │
│ │   Can create a disk from snapshot in ANY region/zone           │
│ │   ⚡ This is how you "move" disks between regions!              │
│ │                                                                 │
│ ├── CONSISTENT: For best consistency:                             │
│ │   ├── Flush disk buffers before snapshot                       │
│ │   ├── Freeze filesystem (fsfreeze) or                          │
│ │   ├── Stop writes briefly or                                   │
│ │   ├── Use VSS snapshots (Windows)                              │
│ │   └── Or just accept crash-consistent (usually fine for        │
│ │       journaled filesystems like ext4/XFS)                     │
│ │                                                                 │
│ ├── STORAGE LOCATION:                                             │
│ │   ├── Multi-regional (default): Stored in the closest          │
│ │   │   multi-region (e.g., "asia" for asia-south1)             │
│ │   ├── Regional: Stored in a specific region                    │
│ │   └── Affects restore speed and cost                          │
│ │                                                                 │
│ └── PRICING:                                                      │
│     ├── First snapshot: $0.026/GB/month (full size)              │
│     └── Incremental: Only pay for changed blocks                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Console Walkthrough — Create Snapshot

```
Console → Compute Engine → Snapshots → Create Snapshot

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE SNAPSHOT                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [webapp-data-snap-20260517]                                  │
│ Description: [Before major migration]                              │
│                                                                       │
│ Source disk: [webapp-data-disk]                                     │
│ Source disk zone: [asia-south1-a]                                  │
│                                                                       │
│ Location:                                                           │
│   ○ Multi-regional (closest to disk) ← default                   │
│   ○ Regional → [select region]                                    │
│                                                                       │
│ Labels:                                                              │
│   created-by: ritesh                                                │
│   purpose: pre-migration                                           │
│                                                                       │
│ Encryption: (inherits from source disk)                            │
│   ⚡ CMEK disks produce CMEK snapshots                              │
│                                                                       │
│ [CREATE]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Snapshot Schedules

```
┌─────────────────────────────────────────────────────────────────────┐
│           SNAPSHOT SCHEDULES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automate snapshots on a recurring schedule.                        │
│                                                                       │
│ Console → Compute Engine → Snapshots → Snapshot Schedules         │
│                                                                       │
│ Name: [daily-backup-schedule]                                      │
│ Region: [asia-south1]                                              │
│                                                                       │
│ Schedule frequency:                                                 │
│   ○ Hourly  → every [N] hours, starting at [HH:MM]               │
│   ○ Daily   → at [02:00] AM                                       │
│   ○ Weekly  → every [Monday] at [02:00]                           │
│                                                                       │
│ Auto-delete snapshots after: [14] days                             │
│   ⚡ Keeps last 14 daily snapshots, auto-deletes older ones       │
│                                                                       │
│ Deletion rule:                                                      │
│   ○ Keep snapshots on schedule deletion                           │
│   ○ Delete snapshots on schedule deletion                         │
│                                                                       │
│ Snapshot labels:                                                    │
│   automated: true                                                   │
│                                                                       │
│ Storage location:                                                   │
│   ○ Multi-regional   ○ Regional                                   │
│                                                                       │
│ ⚡ Attach schedule to disks:                                        │
│   Disk → Edit → Snapshot schedule → select schedule               │
│   A disk can have ONE schedule attached.                           │
│                                                                       │
│ AWS equivalent: EBS Snapshot Lifecycle (via DLM policies)         │
│ Azure equivalent: Azure Backup / Snapshot policies                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Restore from Snapshot

```
┌─────────────────────────────────────────────────────────────────────┐
│           RESTORE FROM SNAPSHOT                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Snapshots create NEW disks — they don't restore in-place.         │
│                                                                       │
│ Use cases:                                                          │
│ ├── Disaster recovery: Create disk from snapshot in new zone     │
│ ├── Cross-region migration: Snapshot in asia → disk in us       │
│ ├── Clone environment: Snapshot prod → create dev disk           │
│ └── Rollback: Before risky change, snapshot → if bad, recreate  │
│                                                                       │
│ Steps:                                                              │
│ 1. Console → Disks → Create Disk                                 │
│ 2. Source type: Snapshot                                           │
│ 3. Select snapshot                                                 │
│ 4. Choose region/zone (can be different from original!)          │
│ 5. Choose disk type (can be different from original!)            │
│ 6. Choose size (must be ≥ snapshot source size)                  │
│ 7. Create → Attach to VM                                         │
│                                                                       │
│ CLI:                                                                │
│ gcloud compute disks create restored-disk \                       │
│   --source-snapshot=webapp-data-snap-20260517 \                   │
│   --zone=us-central1-a \                                          │
│   --type=pd-ssd                                                   │
│                                                                       │
│ ⚡ Lazy restore: Disk is usable immediately, but data is loaded  │
│   in background. First reads to unloaded blocks are slower.      │
│   For full performance: read all blocks after creation.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Machine Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           MACHINE IMAGES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A complete snapshot of a VM instance — includes:             │
│ ├── Boot disk content                                             │
│ ├── All attached data disks                                      │
│ ├── VM configuration (machine type, network, metadata)           │
│ ├── Instance properties & permissions                            │
│ └── Service account configuration                                │
│                                                                       │
│ Think of it as a "VM checkpoint" — everything needed to           │
│ recreate the exact same VM elsewhere.                             │
│                                                                       │
│ Machine Image vs Snapshot vs Custom Image:                         │
│ ┌─────────────────┬─────────┬──────────┬───────────────┐          │
│ │                 │ Machine │ Snapshot │ Custom Image   │          │
│ │                 │ Image   │          │                │          │
│ ├─────────────────┼─────────┼──────────┼───────────────┤          │
│ │ Boot disk       │ ✅      │ ✅       │ ✅             │          │
│ │ Data disks      │ ✅ All  │ ❌ One   │ ❌ One disk    │          │
│ │ VM config       │ ✅      │ ❌       │ ❌             │          │
│ │ Network config  │ ✅      │ ❌       │ ❌             │          │
│ │ Service account │ ✅      │ ❌       │ ❌             │          │
│ │ Incremental     │ ✅      │ ✅       │ ❌ Full copy   │          │
│ │ Cross-region    │ ✅      │ ✅       │ ✅             │          │
│ │ Create VM from  │ ✅      │ ✅ (disk)│ ✅             │          │
│ │ Use in template │ ✅      │ ✅       │ ✅             │          │
│ └─────────────────┴─────────┴──────────┴───────────────┘          │
│                                                                       │
│ Use cases:                                                          │
│ ├── Full VM backup (DR)                                           │
│ ├── Clone a configured VM to another region                      │
│ ├── Create instance template from machine image                  │
│ └── Archive decommissioned VMs (delete VM, keep machine image)  │
│                                                                       │
│ Console → Compute Engine → Machine Images → Create               │
│                                                                       │
│ CLI:                                                                │
│ gcloud compute machine-images create my-vm-backup \               │
│   --source-instance=my-vm \                                       │
│   --source-instance-zone=asia-south1-a \                          │
│   --storage-location=asia                                         │
│                                                                       │
│ # Create VM from machine image:                                   │
│ gcloud compute instances create new-vm \                          │
│   --source-machine-image=my-vm-backup \                           │
│   --zone=us-central1-a                                            │
│                                                                       │
│ AWS equivalent: AMI (but AMI only captures boot + EBS volumes,   │
│   not instance type/network/IAM role configuration)              │
│ Azure equivalent: Azure VM Image / Generalized Image              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Custom Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM IMAGES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A boot disk image that you create from:                      │
│ ├── An existing disk (running or stopped VM's boot disk)         │
│ ├── A snapshot                                                    │
│ ├── Another image                                                │
│ ├── A file in Cloud Storage (raw/tar.gz disk image)             │
│ └── A VM (auto-creates image from its boot disk)                │
│                                                                       │
│ Purpose:                                                            │
│ ├── "Golden image" — pre-configured OS for your org             │
│ ├── Instance templates reference an image for boot disks        │
│ ├── Faster VM startup (everything pre-installed)                │
│ └── Consistency — all VMs start from same base                  │
│                                                                       │
│ Image Families:                                                     │
│ ├── Group related images under a family name                    │
│ ├── "latest" always resolves to newest non-deprecated image     │
│ ├── Use family in instance templates → auto-picks latest        │
│ ├── Can deprecate/obsolete old images in the family             │
│ └── Example: family "webapp-base" → latest = webapp-base-v5     │
│                                                                       │
│ Image states:                                                       │
│ ├── ACTIVE: Available for use                                    │
│ ├── DEPRECATED: Still works, shows warning                      │
│ ├── OBSOLETE: Cannot create new VMs, existing VMs still run     │
│ └── DELETED: Removed                                              │
│                                                                       │
│ Console → Compute Engine → Images → Create Image                  │
│                                                                       │
│ Source:                                                              │
│   ○ Disk → [select boot disk]                                     │
│   ○ Snapshot → [select snapshot]                                  │
│   ○ Cloud Storage file → gs://bucket/image.tar.gz               │
│   ○ Virtual disk (VMDK, VHD) → for migration from on-prem      │
│                                                                       │
│ Family: [webapp-base]   (optional, recommended)                   │
│                                                                       │
│ Location:                                                           │
│   ○ Multi-regional (default)                                      │
│   ○ Regional                                                       │
│                                                                       │
│ CLI:                                                                │
│ # Create image from disk                                          │
│ gcloud compute images create webapp-base-v5 \                     │
│   --source-disk=webapp-vm-boot \                                  │
│   --source-disk-zone=asia-south1-a \                              │
│   --family=webapp-base \                                          │
│   --storage-location=asia                                         │
│                                                                       │
│ # Create image from snapshot                                      │
│ gcloud compute images create webapp-base-v5 \                     │
│   --source-snapshot=webapp-snap-20260517 \                        │
│   --family=webapp-base                                            │
│                                                                       │
│ # Use family in instance template (always gets latest)           │
│ gcloud compute instance-templates create webapp-template \       │
│   --image-family=webapp-base \                                    │
│   --image-project=my-project                                     │
│                                                                       │
│ # Deprecate old image                                             │
│ gcloud compute images deprecate webapp-base-v4 \                 │
│   --state=DEPRECATED \                                            │
│   --replacement=webapp-base-v5                                    │
│                                                                       │
│ AWS equivalent: AMI (Amazon Machine Image)                        │
│ Azure equivalent: Azure Managed Image / Shared Image Gallery     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Local SSD

```
┌─────────────────────────────────────────────────────────────────────┐
│           LOCAL SSD                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Physically attached SSD on the host machine running your VM.│
│       Directly connected via NVMe — EXTREME performance.          │
│                                                                       │
│ ┌─────────────────────────────────────┐                            │
│ │  Physical Host Machine               │                            │
│ │ ┌────────────┐  ┌─────────────────┐ │                            │
│ │ │  VM         │  │  Local SSD       │ │   ← Same physical host   │
│ │ │  Instance   │◄─┤  (NVMe, direct)  │ │   ← NOT network-attached│
│ │ └────────────┘  └─────────────────┘ │                            │
│ │            ▼ network                 │                            │
│ │ ┌─────────────────────────────────┐ │                            │
│ │ │  Persistent Disk (remote)       │ │   ← Network-attached      │
│ │ └─────────────────────────────────┘ │                            │
│ └─────────────────────────────────────┘                            │
│                                                                       │
│ Key properties:                                                     │
│ ├── EPHEMERAL: Data is LOST when:                                │
│ │   ├── VM is stopped (not just restarted — STOPPED)            │
│ │   ├── VM is deleted                                            │
│ │   ├── VM is live-migrated (host maintenance)                  │
│ │   │   ⚡ Set VM to TERMINATE on host maintenance to avoid     │
│ │   │     unexpected data loss during live migration!           │
│ │   └── Host machine has hardware failure                       │
│ │                                                                 │
│ ├── PERFORMANCE:                                                  │
│ │   ├── 900,000 read IOPS / 800,000 write IOPS (per VM max)   │
│ │   ├── 9.4 GB/s read / 6.2 GB/s write throughput              │
│ │   ├── Sub-100μs latency (vs ~1ms for Persistent Disk)        │
│ │   └── ~10-50x faster than Persistent Disk                    │
│ │                                                                 │
│ ├── SIZE: Fixed 375 GB per Local SSD device                      │
│ │   ├── Can attach 1-24 devices (375 GB to 9 TB total)         │
│ │   ├── Number depends on machine type                          │
│ │   └── Cannot resize                                           │
│ │                                                                 │
│ ├── COST: ~$0.048/GB/month (cheaper than pd-ssd per GB)         │
│ │   But can't detach or resize — pay for full allocation       │
│ │                                                                 │
│ ├── NO snapshots, no replication, no backups                     │
│ └── Must be specified at VM creation time (can't add later)    │
│                                                                       │
│ Use cases:                                                          │
│ ├── Caches (Redis, Memcached data directory)                    │
│ ├── Temporary processing (video transcoding, image processing)  │
│ ├── High-performance databases (scratch space, temp tables)     │
│ ├── Swap space                                                   │
│ └── Any workload where data loss is acceptable                 │
│                                                                       │
│ ⚠️ NEVER store critical data on Local SSD without backing it up  │
│   to Persistent Disk or Cloud Storage!                            │
│                                                                       │
│ AWS equivalent: EC2 Instance Store (same concept — ephemeral)    │
│ Azure equivalent: Temporary Disk / Local NVMe (Lsv2/Lsv3 VMs)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a VM with Local SSD

```
Console → Create VM → Advanced → Disks → Add Local SSD

┌─────────────────────────────────────────────────────────────────────┐
│                                                                       │
│ Local SSD interface:                                                │
│   ○ NVMe (recommended — highest performance)                      │
│   ○ SCSI (legacy, slower)                                          │
│                                                                       │
│ Number of Local SSD disks: [4]  (= 4 × 375 GB = 1.5 TB)         │
│                                                                       │
│ ⚡ Available machine types: N1, N2, N2D, C2, C2D, C3, M1, M2, M3 │
│   E2 and T2D do NOT support Local SSD.                            │
│                                                                       │
│ CLI:                                                                │
│ gcloud compute instances create high-perf-vm \                    │
│   --machine-type=n2-standard-8 \                                  │
│   --zone=asia-south1-a \                                          │
│   --local-ssd=interface=NVME \                                    │
│   --local-ssd=interface=NVME \                                    │
│   --local-ssd=interface=NVME \                                    │
│   --local-ssd=interface=NVME                                      │
│   # Each --local-ssd adds one 375 GB device                     │
│                                                                       │
│ Inside VM — format and mount (or RAID):                           │
│ # Single device:                                                   │
│ sudo mkfs.ext4 /dev/nvme0n1                                       │
│ sudo mount /dev/nvme0n1 /mnt/local-ssd                            │
│                                                                       │
│ # Multiple devices — create RAID 0 for max performance:          │
│ sudo mdadm --create /dev/md0 --level=0 --raid-devices=4 \       │
│   /dev/nvme0n1 /dev/nvme0n2 /dev/nvme0n3 /dev/nvme0n4            │
│ sudo mkfs.ext4 /dev/md0                                           │
│ sudo mount /dev/md0 /mnt/local-ssd                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Hyperdisk

```
┌─────────────────────────────────────────────────────────────────────┐
│           HYPERDISK (NEXT-GEN BLOCK STORAGE)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Next-generation Persistent Disk with independently          │
│       scalable IOPS, throughput, and capacity.                    │
│                                                                       │
│ Key difference from Persistent Disk:                                │
│ ├── PD: IOPS scales with disk size (bigger disk = more IOPS)   │
│ ├── Hyperdisk: You set IOPS, throughput, and size INDEPENDENTLY │
│ │   → Don't over-provision disk size just for IOPS!             │
│ └── Dynamic resizing: Change IOPS/throughput without downtime   │
│                                                                       │
│ Hyperdisk types:                                                    │
│ ┌─────────────────┬──────────────┬──────────────┬──────────────┐ │
│ │                 │ Hyperdisk    │ Hyperdisk    │ Hyperdisk    │ │
│ │                 │ Extreme      │ Balanced     │ Throughput   │ │
│ ├─────────────────┼──────────────┼──────────────┼──────────────┤ │
│ │ Max IOPS        │ 350,000      │ 160,000      │ 2,500        │ │
│ │ Max throughput  │ 5,000 MB/s   │ 2,400 MB/s   │ 2,500 MB/s   │ │
│ │ Max size        │ 64 TB        │ 64 TB        │ 32 TB        │ │
│ │ Min size        │ 64 GB        │ 4 GB         │ 2 TB         │ │
│ │ Latency         │ < 100 μs     │ Sub-ms       │ Sub-ms       │ │
│ │ Use case        │ SAP, Oracle, │ General      │ Kafka, big   │ │
│ │                 │ high-perf DB │ purpose ✅    │ data, media  │ │
│ │ VM requirement  │ C3/M3/       │ C3/N2/M3     │ C3/N2/M3     │ │
│ │                 │ A2/A3/X4     │              │              │ │
│ │ Storage pools   │ ✅           │ ✅           │ ✅           │ │
│ └─────────────────┴──────────────┴──────────────┴──────────────┘ │
│                                                                       │
│ Dynamic provisioning:                                               │
│ ├── Change IOPS while disk is attached and VM is running         │
│ ├── Change throughput while disk is running                      │
│ ├── No downtime, no data loss                                    │
│ └── ⚡ Scale up for batch jobs → scale down after → save cost     │
│                                                                       │
│ Hyperdisk Storage Pools:                                           │
│ ├── Pre-provision a pool of capacity + IOPS + throughput        │
│ ├── Create individual disks from the pool                       │
│ ├── Better capacity planning and cost control                   │
│ └── Disks share pool resources                                  │
│                                                                       │
│ AWS equivalent:                                                     │
│ ├── Hyperdisk Extreme ≈ io2 Block Express                       │
│ ├── Hyperdisk Balanced ≈ gp3 (provisioned IOPS)                │
│ └── Hyperdisk Throughput ≈ st1 (but much faster)               │
│                                                                       │
│ Azure equivalent:                                                   │
│ ├── Hyperdisk Extreme ≈ Ultra Disk                              │
│ └── Hyperdisk Balanced ≈ Premium SSD v2                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Disk Encryption

```
┌─────────────────────────────────────────────────────────────────────┐
│           DISK ENCRYPTION                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ALL Persistent Disks are encrypted at rest. Always.               │
│ No option to disable encryption — it's mandatory.                 │
│                                                                       │
│ Three options (same as Cloud Storage):                              │
│                                                                       │
│ 1. GOOGLE-MANAGED (default ✅):                                    │
│    ├── Google creates, manages, rotates the key                  │
│    ├── You do nothing — automatic                                │
│    ├── AES-256 encryption                                        │
│    └── Free, zero configuration                                  │
│                                                                       │
│ 2. CMEK (Customer-Managed Encryption Key):                        │
│    ├── You create a key in Cloud KMS                             │
│    ├── GCP uses YOUR key to encrypt the disk                    │
│    ├── You control rotation, can disable/revoke                 │
│    ├── ⚠️ If you destroy the key → disk data is UNRECOVERABLE   │
│    ├── Audit key usage via Cloud Audit Logs                     │
│    ├── Snapshots of CMEK disks are also CMEK-encrypted         │
│    └── Cost: KMS key fees ($0.06/month + per-operation)        │
│                                                                       │
│    Grant the Compute Engine service agent access to the key:    │
│    gcloud kms keys add-iam-policy-binding KEY_NAME \            │
│      --keyring=KEY_RING \                                        │
│      --location=LOCATION \                                       │
│      --member=serviceAccount:                                    │
│        service-PROJECT_NUMBER@compute-system.iam.gserviceaccount│
│        .com \                                                    │
│      --role=roles/cloudkms.cryptoKeyEncrypterDecrypter           │
│                                                                       │
│ 3. CSEK (Customer-Supplied Encryption Key):                       │
│    ├── YOU provide a raw 256-bit AES key or RSA-wrapped key     │
│    ├── Google uses it to encrypt, does NOT store it             │
│    ├── You must provide the key for EVERY operation             │
│    │   (create, attach, snapshot, resize, etc.)                 │
│    ├── ⚠️ If you lose the key → data is GONE                    │
│    ├── Cannot use Console — CLI/API only                        │
│    └── Operationally complex — rarely used in practice         │
│                                                                       │
│ AWS: SSE-S3 / SSE-KMS / SSE-C  (same 3 models)                  │
│ Azure: Platform-managed / Customer-managed / Customer-provided   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Performance Tuning

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERFORMANCE TUNING GUIDE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Disk performance depends on THREE things:                          │
│ 1. Disk type (pd-standard < pd-balanced < pd-ssd < pd-extreme)  │
│ 2. Disk size (bigger disk = more IOPS, up to cap)               │
│ 3. VM machine type (small VMs cap disk IOPS!)                   │
│                                                                       │
│ ⚡ THE BOTTLENECK IS USUALLY THE VM, NOT THE DISK.                 │
│                                                                       │
│ Per-VM IOPS limits (examples):                                     │
│ ┌─────────────────────┬──────────────┬──────────────┐             │
│ │ Machine type        │ Max read     │ Max write    │             │
│ │                     │ IOPS         │ IOPS         │             │
│ ├─────────────────────┼──────────────┼──────────────┤             │
│ │ e2-micro            │ 1,500        │ 1,500        │             │
│ │ e2-standard-4       │ 15,000       │ 15,000       │             │
│ │ n2-standard-8       │ 30,000       │ 30,000       │             │
│ │ n2-standard-16      │ 60,000       │ 30,000       │             │
│ │ n2-standard-32+     │ 100,000      │ 60,000       │             │
│ │ c3-standard-44      │ 160,000      │ 90,000       │             │
│ │ c3-highmem-176      │ 350,000      │ 350,000      │             │
│ └─────────────────────┴──────────────┴──────────────┘             │
│                                                                       │
│ ⚡ A 10 TB pd-ssd gives 300K IOPS potential, but an e2-standard-4│
│   VM caps at 15K IOPS. You'd waste 95% of the disk's capacity!  │
│                                                                       │
│ TUNING TIPS:                                                        │
│ ├── Match disk type to workload:                                 │
│ │   ├── Logs/archival → pd-standard (cheapest, IOPS doesn't    │
│ │   │   matter)                                                  │
│ │   ├── General/boot → pd-balanced (good price/performance)    │
│ │   ├── Database → pd-ssd or pd-extreme                        │
│ │   └── Extreme → Hyperdisk Extreme or Local SSD               │
│ │                                                                 │
│ ├── If IOPS is too low:                                          │
│ │   1. Check if VM is the bottleneck (VM IOPS limit)           │
│ │   2. Increase disk size (IOPS scales with size)              │
│ │   3. Upgrade disk type (pd-balanced → pd-ssd)                │
│ │   4. Use pd-extreme or Hyperdisk (provision IOPS directly)   │
│ │                                                                 │
│ ├── If throughput is too low:                                    │
│ │   Same approach — scale disk size or upgrade type             │
│ │                                                                 │
│ ├── Use RAID 0 with multiple disks for more throughput          │
│ │   (rarely needed with Hyperdisk)                              │
│ │                                                                 │
│ └── Monitor: Cloud Monitoring → VM → Disk metrics              │
│     ├── disk/read_ops_count                                     │
│     ├── disk/write_ops_count                                    │
│     ├── disk/read_bytes_count                                   │
│     └── disk/write_bytes_count                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Terraform & CLI

### Terraform

```hcl
# === Persistent Disk ===

# pd-balanced data disk
resource "google_compute_disk" "data" {
  name  = "webapp-data-disk"
  type  = "pd-balanced"
  zone  = "asia-south1-a"
  size  = 200  # GB

  labels = {
    environment = "prod"
    team        = "backend"
  }
}

# Regional pd-ssd (HA — replicated across 2 zones)
resource "google_compute_region_disk" "ha_data" {
  name          = "db-ha-disk"
  type          = "pd-ssd"
  region        = "asia-south1"
  size          = 500
  replica_zones = [
    "asia-south1-a",
    "asia-south1-b",
  ]
}

# pd-extreme with provisioned IOPS
resource "google_compute_disk" "extreme" {
  name             = "oracle-db-disk"
  type             = "pd-extreme"
  zone             = "asia-south1-a"
  size             = 1000
  provisioned_iops = 80000
}

# CMEK-encrypted disk
resource "google_compute_disk" "encrypted" {
  name = "encrypted-data-disk"
  type = "pd-balanced"
  zone = "asia-south1-a"
  size = 100

  disk_encryption_key {
    kms_key_self_link = google_kms_crypto_key.disk_key.id
  }
}

# Attach data disk to VM
resource "google_compute_attached_disk" "data_attach" {
  disk     = google_compute_disk.data.id
  instance = google_compute_instance.webapp.id
  mode     = "READ_WRITE"  # or READ_ONLY for multi-attach
}

# VM with Local SSD
resource "google_compute_instance" "high_perf" {
  name         = "high-perf-vm"
  machine_type = "n2-standard-8"
  zone         = "asia-south1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      type  = "pd-balanced"
      size  = 50
    }
  }

  # Local SSD #1
  scratch_disk {
    interface = "NVME"
  }

  # Local SSD #2
  scratch_disk {
    interface = "NVME"
  }

  network_interface {
    network    = "default"
    subnetwork = "default"
  }

  # Set to TERMINATE for Local SSD data preservation awareness
  scheduling {
    on_host_maintenance = "TERMINATE"
    automatic_restart   = true
  }
}

# === Snapshots ===

# One-time snapshot
resource "google_compute_snapshot" "data_snap" {
  name        = "webapp-data-snap-20260517"
  source_disk = google_compute_disk.data.id
  zone        = "asia-south1-a"

  storage_locations = ["asia"]

  labels = {
    purpose = "pre-migration"
  }
}

# Snapshot schedule (automated daily backups)
resource "google_compute_resource_policy" "daily_backup" {
  name   = "daily-backup-schedule"
  region = "asia-south1"

  snapshot_schedule_policy {
    schedule {
      daily_schedule {
        days_in_cycle = 1
        start_time    = "02:00"
      }
    }

    retention_policy {
      max_retention_days    = 14
      on_source_disk_delete = "KEEP_AUTO_SNAPSHOTS"
    }

    snapshot_properties {
      labels = {
        automated = "true"
      }
      storage_locations = ["asia"]
    }
  }
}

# Attach snapshot schedule to disk
resource "google_compute_disk_resource_policy_attachment" "attach_schedule" {
  name = google_compute_resource_policy.daily_backup.name
  disk = google_compute_disk.data.name
  zone = "asia-south1-a"
}

# Create disk from snapshot (restore)
resource "google_compute_disk" "restored" {
  name     = "webapp-data-restored"
  type     = "pd-ssd"              # Can change type on restore
  zone     = "us-central1-a"       # Can restore in different region
  size     = 200                    # Must be >= source size
  snapshot = google_compute_snapshot.data_snap.id
}

# === Custom Image ===

resource "google_compute_image" "webapp_base" {
  name        = "webapp-base-v5"
  family      = "webapp-base"
  source_disk = google_compute_disk.golden_disk.id

  labels = {
    version = "5"
    os      = "debian-12"
  }
}

# === Machine Image ===

resource "google_compute_machine_image" "vm_backup" {
  name            = "webapp-vm-backup-20260517"
  source_instance = google_compute_instance.webapp.id
}
```

### gcloud CLI Reference

```bash
# =============================================
# PERSISTENT DISK
# =============================================

# Create a balanced disk
gcloud compute disks create webapp-data \
  --type=pd-balanced \
  --size=200GB \
  --zone=asia-south1-a

# Create regional disk (HA)
gcloud compute disks create db-ha-disk \
  --type=pd-ssd \
  --size=500GB \
  --region=asia-south1 \
  --replica-zones=asia-south1-a,asia-south1-b

# Create extreme disk with provisioned IOPS
gcloud compute disks create oracle-disk \
  --type=pd-extreme \
  --size=1000GB \
  --provisioned-iops=80000 \
  --zone=asia-south1-a

# List disks
gcloud compute disks list --filter="zone:asia-south1-a"

# Describe disk (see IOPS, type, status)
gcloud compute disks describe webapp-data --zone=asia-south1-a

# Resize disk (increase only, no downtime)
gcloud compute disks resize webapp-data \
  --size=500GB \
  --zone=asia-south1-a

# Attach disk to running VM
gcloud compute instances attach-disk webapp-vm \
  --disk=webapp-data \
  --zone=asia-south1-a \
  --mode=rw

# Detach disk from VM
gcloud compute instances detach-disk webapp-vm \
  --disk=webapp-data \
  --zone=asia-south1-a

# Delete disk
gcloud compute disks delete webapp-data --zone=asia-south1-a

# =============================================
# SNAPSHOTS
# =============================================

# Create snapshot
gcloud compute snapshots create webapp-snap-20260517 \
  --source-disk=webapp-data \
  --source-disk-zone=asia-south1-a \
  --storage-location=asia

# List snapshots
gcloud compute snapshots list

# Create disk from snapshot (in different region)
gcloud compute disks create webapp-restored \
  --source-snapshot=webapp-snap-20260517 \
  --zone=us-central1-a \
  --type=pd-ssd

# Create snapshot schedule
gcloud compute resource-policies create snapshot-schedule daily-backup \
  --region=asia-south1 \
  --max-retention-days=14 \
  --daily-schedule \
  --start-time=02:00 \
  --storage-location=asia

# Attach schedule to disk
gcloud compute disks add-resource-policies webapp-data \
  --resource-policies=daily-backup \
  --zone=asia-south1-a

# Delete snapshot
gcloud compute snapshots delete webapp-snap-20260517

# =============================================
# IMAGES
# =============================================

# Create image from disk
gcloud compute images create webapp-base-v5 \
  --source-disk=golden-disk \
  --source-disk-zone=asia-south1-a \
  --family=webapp-base

# Create image from snapshot
gcloud compute images create webapp-base-v5 \
  --source-snapshot=golden-snap

# List images in a family
gcloud compute images list --filter="family=webapp-base"

# Get latest image in family
gcloud compute images describe-from-family webapp-base

# Deprecate old image
gcloud compute images deprecate webapp-base-v4 \
  --state=DEPRECATED \
  --replacement=webapp-base-v5

# =============================================
# MACHINE IMAGES
# =============================================

# Create machine image (full VM capture)
gcloud compute machine-images create webapp-backup \
  --source-instance=webapp-vm \
  --source-instance-zone=asia-south1-a \
  --storage-location=asia

# Create VM from machine image
gcloud compute instances create webapp-clone \
  --source-machine-image=webapp-backup \
  --zone=us-central1-a

# List machine images
gcloud compute machine-images list

# =============================================
# LOCAL SSD
# =============================================

# Create VM with 4 Local SSD devices (4 × 375 GB = 1.5 TB)
gcloud compute instances create local-ssd-vm \
  --machine-type=n2-standard-8 \
  --zone=asia-south1-a \
  --local-ssd=interface=NVME \
  --local-ssd=interface=NVME \
  --local-ssd=interface=NVME \
  --local-ssd=interface=NVME

# =============================================
# HYPERDISK
# =============================================

# Create Hyperdisk Balanced
gcloud compute disks create hyper-balanced \
  --type=hyperdisk-balanced \
  --size=100GB \
  --provisioned-iops=10000 \
  --provisioned-throughput=300 \
  --zone=asia-south1-a

# Update IOPS dynamically (no downtime)
gcloud compute disks update hyper-balanced \
  --provisioned-iops=20000 \
  --zone=asia-south1-a
```

---

## Part 13: Real-World Patterns

### Startup

```
┌─────────────────────────────────────────────────────────────────────┐
│ STARTUP (5-10 developers)                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Disks:                                                               │
│ ├── pd-balanced for everything (boot + data)                     │
│ ├── 50 GB boot, 100-200 GB data disks                            │
│ ├── Single zone (save cost, accept zone-failure risk)            │
│ └── Resize as needed (start small, grow)                         │
│                                                                       │
│ Backups:                                                             │
│ ├── Daily snapshot schedule (14-day retention)                   │
│ ├── Manual snapshot before risky changes                         │
│ └── Google-managed encryption (free, simple)                     │
│                                                                       │
│ Images:                                                              │
│ ├── One custom image family for standard VM setup                │
│ └── Update monthly with latest patches                           │
│                                                                       │
│ Local SSD: Not needed                                               │
│ Hyperdisk: Not needed                                               │
│                                                                       │
│ Monthly cost example:                                               │
│   3 VMs × (50 GB boot + 200 GB data) × $0.10/GB                 │
│   = $75/month for all disks                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Mid-Size

```
┌─────────────────────────────────────────────────────────────────────┐
│ MID-SIZE (50-100 developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Disks:                                                               │
│ ├── pd-balanced for general workloads                            │
│ ├── pd-ssd for databases (MySQL, PostgreSQL)                    │
│ ├── Regional disks for database servers (HA)                    │
│ └── pd-standard for log storage and cold data                   │
│                                                                       │
│ Backups:                                                             │
│ ├── Hourly snapshots for DB disks (7-day retention)             │
│ ├── Daily snapshots for app disks (30-day retention)            │
│ ├── Cross-region snapshot copies for DR                          │
│ └── CMEK for database disks (compliance)                        │
│                                                                       │
│ Images:                                                              │
│ ├── Image families per team (webapp-base, api-base, worker-base)│
│ ├── Image build pipeline (Packer or Cloud Build)                │
│ └── Weekly image updates with security patches                  │
│                                                                       │
│ Machine Images:                                                      │
│ ├── Monthly machine image of critical VMs                       │
│ └── Before major version upgrades                                │
│                                                                       │
│ Local SSD: For Redis cache servers                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Enterprise

```
┌─────────────────────────────────────────────────────────────────────┐
│ ENTERPRISE (500+ developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Disks:                                                               │
│ ├── Hyperdisk Balanced for standard workloads                    │
│ │   (dynamic IOPS provisioning — scale up/down as needed)       │
│ ├── Hyperdisk Extreme for SAP HANA / Oracle / mission-critical  │
│ ├── Regional disks for all database servers                     │
│ ├── pd-standard for archival and compliance logs                │
│ └── Hyperdisk Storage Pools for capacity planning               │
│                                                                       │
│ Backups:                                                             │
│ ├── Continuous snapshots (hourly for critical, daily for others) │
│ ├── Cross-region snapshot replication (DR strategy)              │
│ ├── Tested restore procedures (quarterly DR drills)             │
│ └── Machine images for complex multi-disk VMs                   │
│                                                                       │
│ Encryption:                                                         │
│ ├── CMEK mandatory for all production disks                     │
│ ├── Separate KMS keys per data classification                   │
│ ├── Key rotation every 90 days                                   │
│ └── Cloud Audit Logs for all key access                         │
│                                                                       │
│ Images:                                                              │
│ ├── Shared image project (org-wide golden images)               │
│ ├── CIS-hardened base images                                     │
│ ├── Automated pipeline: build → scan → test → promote          │
│ ├── Image families with strict deprecation policies             │
│ └── Compliance: only approved images can be used (Org Policy)  │
│                                                                       │
│ Local SSD: For in-memory databases, caches, ML training scratch │
│                                                                       │
│ Monitoring:                                                         │
│ ├── Disk IOPS and throughput alerts                              │
│ ├── Snapshot job failure alerts                                  │
│ └── KMS key expiration alerts                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌──────────────────────────────────────────────────────────────────────┐
│ PERSISTENT DISK & LOCAL SSD — QUICK REFERENCE                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ Disk types (Persistent Disk):                                        │
│ ├── pd-standard (HDD): Cheapest, 7.5K read IOPS, logs/archive     │
│ ├── pd-balanced (SSD): Best value ✅, 80K read IOPS, general use   │
│ ├── pd-ssd (SSD): High perf, 100K read IOPS, databases            │
│ └── pd-extreme (SSD): Highest, 120K IOPS (provisioned), SAP/Oracle│
│                                                                        │
│ Hyperdisk:                                                            │
│ ├── Extreme: 350K IOPS, independent IOPS/throughput/size            │
│ ├── Balanced: 160K IOPS, general purpose next-gen                   │
│ └── Throughput: 2.5 GB/s, big data/streaming                        │
│                                                                        │
│ Local SSD: 375 GB × N devices, 900K IOPS, EPHEMERAL ⚠️              │
│                                                                        │
│ Key rules:                                                            │
│ ├── Can increase size — NEVER decrease                               │
│ ├── Can't change disk type (use snapshot to migrate)                │
│ ├── IOPS scales with size (except pd-extreme & Hyperdisk)           │
│ ├── VM machine type caps disk IOPS                                   │
│ ├── Regional disk = 2x cost, sync replication, zone-HA             │
│ ├── Multi-attach = read-only only (max 10 VMs)                      │
│ └── Max 128 disks per VM                                             │
│                                                                        │
│ Snapshots: Incremental, global, cross-region restore                 │
│ Machine Image: Full VM capture (all disks + config)                  │
│ Custom Image: Boot disk only, use families for versioning            │
│                                                                        │
│ Encryption: Google-managed (free) | CMEK (KMS) | CSEK (you supply) │
│                                                                        │
│ AWS ↔ GCP mapping:                                                     │
│ ├── EBS gp3 → pd-balanced                                            │
│ ├── EBS io2 → pd-extreme / Hyperdisk Extreme                        │
│ ├── EBS st1 → pd-standard                                            │
│ ├── Instance Store → Local SSD                                        │
│ ├── EBS Snapshot → PD Snapshot                                        │
│ ├── AMI → Custom Image + Machine Image                                │
│ └── DLM → Snapshot Schedule                                           │
│                                                                        │
│ Azure ↔ GCP mapping:                                                   │
│ ├── Premium SSD → pd-ssd                                              │
│ ├── Standard SSD → pd-balanced                                        │
│ ├── Ultra Disk → Hyperdisk Extreme                                    │
│ ├── Premium SSD v2 → Hyperdisk Balanced                               │
│ ├── Temp Disk → Local SSD                                              │
│ ├── Managed Disk Snapshot → PD Snapshot                                │
│ └── Shared Image Gallery → Image Families                              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## What is IOPS? (Beginner Explanation)

```
┌─────────────────────────────────────────────────────────────────────┐
│           IOPS — INPUT/OUTPUT OPERATIONS PER SECOND                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ IOPS = How many read/write operations your disk can do per second. │
│                                                                       │
│ 🚗 Highway Analogy:                                                 │
│ ├── Think of your disk as a highway                              │
│ ├── Each car on the highway = one read or write operation        │
│ ├── IOPS = how many cars can pass through per second             │
│ ├── More lanes (higher IOPS) = more cars at once                │
│ └── Throughput (MB/s) = how BIG each car is (data size per op)  │
│                                                                       │
│ Simple example:                                                     │
│ ├── pd-standard: 7,500 read IOPS → 7,500 small reads/second    │
│ │   (like a 2-lane road — fine for light traffic)               │
│ ├── pd-ssd: 100,000 read IOPS → 100,000 small reads/second     │
│ │   (like a 12-lane highway — handles rush hour)                │
│ └── pd-extreme: 120,000 read IOPS → maximum speed!              │
│     (like an autobahn — built for heavy traffic)                │
│                                                                       │
│ ── WHY IOPS MATTERS FOR DATABASES ──                                │
│                                                                       │
│ Databases do LOTS of small, random reads and writes:               │
│ ├── Each SQL query = multiple disk reads (index lookups, rows)  │
│ ├── Each INSERT/UPDATE = disk writes (data + write-ahead log)   │
│ ├── 100 users doing queries = hundreds of IOPS needed           │
│ ├── If IOPS is too low → queries queue up → slow response      │
│ └── This is why databases need pd-ssd or pd-extreme, not HDD   │
│                                                                       │
│ Quick rule of thumb:                                                │
│ ├── Logs, backups, archives → IOPS doesn't matter (pd-standard)│
│ ├── Web servers, general apps → moderate IOPS (pd-balanced)     │
│ ├── Databases (MySQL, Postgres) → high IOPS (pd-ssd)           │
│ └── Heavy OLTP, SAP, Oracle → extreme IOPS (pd-extreme)        │
│                                                                       │
│ ⚡ Remember: In GCP, IOPS scales with disk SIZE (except pd-extreme)│
│   Want more IOPS? Make the disk bigger!                            │
│   100 GB pd-balanced = 600 IOPS                                   │
│   1 TB pd-balanced = 6,000 IOPS                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Disks

### Detaching a Disk from a VM

```
Console → Compute Engine → VM Instances → [click your VM]
→ Edit

┌─────────────────────────────────────────────────────────────────────┐
│           DETACH DISK FROM VM                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scroll to "Additional disks" section:                              │
│                                                                       │
│ ┌──────────────┬──────────┬──────────────┬─────────┐               │
│ │ Name         │ Type     │ Mode         │ Action  │               │
│ ├──────────────┼──────────┼──────────────┼─────────┤               │
│ │ data-disk-1  │ pd-ssd   │ Read/Write   │ [✕]     │ ← click ✕    │
│ └──────────────┴──────────┴──────────────┴─────────┘               │
│                                                                       │
│ Click the ✕ (remove) button next to the disk you want to detach.  │
│ Then click [Save] at the bottom.                                   │
│                                                                       │
│ ⚡ The VM usually needs to be STOPPED to detach a disk safely.     │
│   (Some OS/configurations allow hot-detach, but stopping is safer)│
│ ⚡ Detaching does NOT delete the disk — it remains in your project │
│   under Compute Engine → Disks, ready to reattach to another VM. │
│ ⚡ Always unmount the filesystem INSIDE the VM first (sudo umount) │
│   before detaching, to avoid data corruption.                      │
│                                                                       │
│ Alternative path:                                                   │
│ Console → Compute Engine → Disks → [click disk] → the "In use by"│
│ section shows which VM it's attached to.                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Deleting a Disk

```
Console → Compute Engine → Disks → [select disk checkbox]
→ Delete (top bar)

┌─────────────────────────────────────────────────────────────────────┐
│           DELETE DISK                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️  "Are you sure you want to delete 'data-disk-1'?"                │
│                                                                       │
│ Prerequisites:                                                      │
│ ├── Disk must NOT be attached to any VM                           │
│ │   ⚠️ If attached, detach it first (see above)                   │
│ └── Disk must not have any active snapshot operations             │
│                                                                       │
│ What happens:                                                       │
│ ├── All data on the disk is PERMANENTLY deleted                  │
│ ├── This action CANNOT be undone                                  │
│ ├── Existing snapshots of the disk are NOT deleted               │
│ │   (snapshots are independent resources)                        │
│ └── Any snapshot schedule attached to the disk is NOT deleted    │
│                                                                       │
│ ⚡ Take a snapshot before deleting if you might need the data!     │
│ ⚡ Boot disks can only be deleted after the VM is deleted           │
│   (unless "delete boot disk" was unchecked during VM creation).  │
│                                                                       │
│ [Delete]   [Cancel]                                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Resizing a Disk (Increase Only)

```
Console → Compute Engine → Disks → [click your disk]
→ Edit (pencil icon at top)

┌─────────────────────────────────────────────────────────────────────┐
│           RESIZE DISK                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Size (GB): [100] → change to [200]                                 │
│                                                                       │
│ [Save]                                                               │
│                                                                       │
│ Rules:                                                               │
│ ├── ✅ Can INCREASE size at any time (even while VM is running)    │
│ ├── ❌ Can NEVER decrease size — once bigger, always bigger!       │
│ ├── No VM restart required (online resize)                        │
│ └── New size takes effect in seconds on the GCP side              │
│                                                                       │
│ ⚠️  IMPORTANT — You're not done yet!                                 │
│ After resizing the disk in Console, you must also resize the      │
│ filesystem INSIDE the VM so the OS can see the new space:         │
│                                                                       │
│   # SSH into the VM, then:                                        │
│   # For ext4:                                                      │
│   sudo resize2fs /dev/sdb                                          │
│                                                                       │
│   # For XFS:                                                       │
│   sudo xfs_growfs /mnt/data                                       │
│                                                                       │
│   # Verify:                                                        │
│   df -h /mnt/data                                                  │
│                                                                       │
│ ⚡ If you need a SMALLER disk: Snapshot → Create new smaller disk  │
│   → Copy data → Delete old disk. There's no direct shrink.       │
│ ⚡ If you need to change DISK TYPE (e.g., pd-standard → pd-ssd):  │
│   Same approach — Snapshot → Create new disk of desired type.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Disk from a Snapshot

```
Console → Compute Engine → Disks → Create Disk

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE DISK FROM SNAPSHOT                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [restored-data-disk]                                         │
│                                                                       │
│ Disk source type: ● Snapshot                                       │
│ Source snapshot: [webapp-data-snap-20260517 ▼]                    │
│   ⚡ Dropdown shows all snapshots in the project.                  │
│   ⚡ Snapshots are global — you can restore in ANY region/zone.   │
│                                                                       │
│ Location:                                                           │
│   Region: [asia-south1]   (can be different from source!)        │
│   Zone: [asia-south1-a]                                            │
│   ⚡ This is how you "move" disks between regions — snapshot in   │
│     one region, restore in another.                                │
│                                                                       │
│ Disk type: [Balanced persistent disk ▼]                           │
│   ⚡ Can choose a DIFFERENT type than the original disk!           │
│   Example: Source was pd-standard → restore as pd-ssd for         │
│   better performance.                                              │
│                                                                       │
│ Size (GB): [100]                                                   │
│   ⚡ Must be >= snapshot size. Can make it larger.                 │
│   ⚡ Cannot make it smaller than the original disk.                │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ After creation:                                                     │
│ 1. Attach the new disk to a VM (Edit VM → Additional disks)      │
│ 2. Mount it (sudo mount /dev/sdb /mnt/data)                      │
│ 3. Verify data is intact                                          │
│                                                                       │
│ ⚡ The restored disk is a completely independent copy — changes    │
│   to it do NOT affect the snapshot or the original disk.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 23: Filestore** → `23-filestore.md`
