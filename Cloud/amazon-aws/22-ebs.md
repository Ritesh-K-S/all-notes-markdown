# Chapter 22: EBS - Elastic Block Store

---

## Table of Contents

- [Overview](#overview)
- [Part 1: EBS Fundamentals](#part-1-ebs-fundamentals)
- [Part 2: Volume Types](#part-2-volume-types)
- [Part 3: Creating an EBS Volume (Full Portal Walkthrough)](#part-3-creating-an-ebs-volume-full-portal-walkthrough)
- [Part 4: Attaching & Mounting Volumes](#part-4-attaching--mounting-volumes)
- [Part 5: Snapshots](#part-5-snapshots)
- [Part 6: Encryption](#part-6-encryption)
- [Part 7: Multi-Attach](#part-7-multi-attach)
- [Part 8: Instance Store (Ephemeral Storage)](#part-8-instance-store-ephemeral-storage)
- [Part 9: Performance Optimization](#part-9-performance-optimization)
- [Part 10: Terraform & CLI Examples](#part-10-terraform--cli-examples)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is EBS? Why Do We Need It?

When you buy a laptop, it comes with a hard drive (SSD/HDD) to store the operating system and your files. Similarly, every EC2 instance (virtual server) in AWS needs a disk — that disk is an **EBS volume**.

**Think of EBS as a virtual hard drive** that you plug into your EC2 instance. The key difference from a physical hard drive: if your EC2 instance crashes, you can detach the EBS volume and attach it to a new instance — your data survives. It's like a USB external drive for the cloud.

**Why not just use the instance's built-in storage?**
- EC2 instances have optional "Instance Store" storage, but it's **temporary** — when the instance stops, all data is wiped
- EBS volumes are **persistent** — data stays even when the instance is stopped or terminated (if configured)
- EBS volumes can be **backed up** (snapshots) and **encrypted**

**Simple real-world examples:**
- 💻 The root volume of every EC2 instance (contains the OS) is an EBS volume
- 🗄️ A database server stores its data files on a high-performance EBS volume
- 📦 You take a snapshot of your EBS volume before making risky changes (like an undo button)

Amazon EBS provides persistent block storage volumes for EC2 instances. It persists independently from the life of the instance.

```
What you'll learn:
├── EBS Fundamentals
│   ├── Block storage vs object storage vs file storage
│   ├── AZ-scoped (not region-scoped)
│   └── Pricing model
├── Volume Types
│   ├── gp3 (General Purpose SSD) — default, best value
│   ├── gp2 (General Purpose SSD) — legacy
│   ├── io2/io2 Block Express (Provisioned IOPS SSD)
│   ├── st1 (Throughput Optimized HDD)
│   └── sc1 (Cold HDD)
├── Creating an EBS Volume (Full Portal Walkthrough)
├── Attaching & Mounting Volumes
├── Snapshots (backups)
├── Encryption
├── Multi-Attach (io2 only)
├── Instance Store (ephemeral NVMe)
├── Performance Optimization
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: EBS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           EBS CORE CONCEPTS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is EBS?                                                         │
│ ├── Block storage = virtual hard drive                             │
│ ├── Attached to EC2 instances over the network                    │
│ ├── Persists independently of EC2 instance lifecycle              │
│ ├── Can be detached from one instance, attached to another       │
│ ├── One volume = one AZ (cannot span AZs)                        │
│ └── Root volume: OS disk. Data volumes: additional storage.       │
│                                                                       │
│ Storage types comparison:                                            │
│ ┌──────────────┬──────────────┬──────────────┬───────────────────┐ │
│ │ Type         │ Block (EBS)  │ Object (S3)  │ File (EFS)        │ │
│ ├──────────────┼──────────────┼──────────────┼───────────────────┤ │
│ │ Access       │ Single EC2   │ HTTP/S API   │ Multiple EC2      │ │
│ │ Pattern      │ instance     │ (any client) │ instances         │ │
│ │ Use case     │ OS disk,     │ Backups,     │ Shared file       │ │
│ │              │ databases    │ static files │ systems, CMS      │ │
│ │ Performance  │ Low latency  │ Varies       │ Low-moderate      │ │
│ │ Cost         │ $/GB/month   │ $/GB/month   │ $/GB/month        │ │
│ │ Scope        │ Single AZ    │ Region       │ Region (multi-AZ) │ │
│ └──────────────┴──────────────┴──────────────┴───────────────────┘ │
│                                                                       │
│ Key characteristics:                                                 │
│ ├── Network-attached (not physically on the host)                │
│ ├── Can be detached/re-attached (non-root volumes)               │
│ ├── Snapshots: point-in-time backup to S3                        │
│ ├── Encryption: AES-256, transparent to EC2                      │
│ ├── Delete on termination: root=yes, data volumes=no (default)   │
│ └── One volume → one instance (except io2 Multi-Attach)          │
│                                                                       │
│ Pricing (us-east-1, approximate):                                   │
│ ├── gp3: $0.08/GB/month + IOPS/throughput if above baseline     │
│ ├── gp2: $0.10/GB/month (burstable IOPS)                        │
│ ├── io2: $0.125/GB/month + $0.065/provisioned IOPS              │
│ ├── st1: $0.045/GB/month                                         │
│ ├── sc1: $0.015/GB/month                                         │
│ └── Snapshots: $0.05/GB/month (incremental)                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Volume Types

```
┌─────────────────────────────────────────────────────────────────────┐
│           EBS VOLUME TYPES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌───────────┬───────┬────────────┬────────────┬───────────────────┐ │
│ │ Type      │ Media │ IOPS       │ Throughput │ Use Case          │ │
│ ├───────────┼───────┼────────────┼────────────┼───────────────────┤ │
│ │ gp3       │ SSD   │ 3,000 base │ 125 MB/s   │ Most workloads   │ │
│ │ ⚡Default │       │ up to 16K  │ up to 1GB/s│ Boot, dev, prod  │ │
│ │           │       │            │            │ Best price/perf   │ │
│ │           │       │            │            │                   │ │
│ │ gp2       │ SSD   │ 3 IOPS/GB │ up to      │ Legacy — use gp3 │ │
│ │ (legacy)  │       │ burst: 3K  │ 250 MB/s   │ instead          │ │
│ │           │       │ max: 16K   │            │                   │ │
│ │           │       │            │            │                   │ │
│ │ io2       │ SSD   │ up to 64K  │ up to 1GB/s│ Critical DBs     │ │
│ │ Prov IOPS │       │ (256K BE)  │ (4GB/s BE) │ Latency-sensitive│ │
│ │           │       │            │            │ 99.999% durable  │ │
│ │           │       │            │            │                   │ │
│ │ st1       │ HDD   │ 500 IOPS   │ 500 MB/s   │ Big data, logs   │ │
│ │ Throughput│       │ (baseline) │ (baseline) │ Data warehousing  │ │
│ │           │       │            │            │ Sequential reads  │ │
│ │           │       │            │            │                   │ │
│ │ sc1       │ HDD   │ 250 IOPS   │ 250 MB/s   │ Archive, cold    │ │
│ │ Cold      │       │ (baseline) │ (baseline) │ Lowest cost      │ │
│ │           │       │            │            │ Infrequent access │ │
│ └───────────┴───────┴────────────┴────────────┴───────────────────┘ │
│                                                                       │
│ gp3 vs gp2 — why gp3 is better:                                   │
│ ├── gp3: Fixed 3,000 IOPS baseline regardless of size            │
│ ├── gp2: IOPS scales with size (3 IOPS/GB, need 1TB for 3K)    │
│ ├── gp3: 20% cheaper per GB ($0.08 vs $0.10)                    │
│ ├── gp3: Independently configure IOPS and throughput             │
│ └── ⚡ Always use gp3 for new volumes                             │
│                                                                       │
│ Decision guide:                                                      │
│ ├── "Default / most workloads" → gp3                             │
│ ├── "Database needing >16K IOPS" → io2                           │
│ ├── "Need guaranteed sub-ms latency" → io2                       │
│ ├── "Big data, sequential throughput" → st1                      │
│ ├── "Archive, lowest cost, infrequent" → sc1                    │
│ └── "Boot volume" → gp3 (SSD required for boot)                 │
│                                                                       │
│ ⚠️ HDD volumes (st1, sc1) CANNOT be used as boot volumes.         │
│ Only SSD volumes (gp2, gp3, io1, io2) can be boot volumes.       │
│                                                                       │
│ io2 Block Express:                                                   │
│ ├── Next-gen io2 for Nitro-based instances                       │
│ ├── Up to 256,000 IOPS (vs 64K regular io2)                    │
│ ├── Up to 4,000 MB/s throughput                                  │
│ ├── Up to 64 TB volume size                                      │
│ ├── Sub-millisecond latency                                      │
│ └── For: SAP HANA, Oracle, SQL Server, mission-critical DBs    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating an EBS Volume (Full Portal Walkthrough)

```
Console → EC2 → Elastic Block Store → Volumes → Create volume

┌─────────────────────────────────────────────────────────────────┐
│           CREATE VOLUME                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Volume settings ──                                           │
│                                                                   │
│ Volume type: [General Purpose SSD (gp3) ▼]                    │
│   ├── General Purpose SSD (gp3)    ← ⚡ Best default          │
│   ├── General Purpose SSD (gp2)    ← Legacy                  │
│   ├── Provisioned IOPS SSD (io2)   ← High performance        │
│   ├── Provisioned IOPS SSD (io1)   ← Legacy io               │
│   ├── Throughput Optimized HDD (st1) ← Sequential reads      │
│   └── Cold HDD (sc1)               ← Cheapest                │
│                                                                   │
│ Size (GiB): [100]                                              │
│ → gp3: 1 GiB - 16,384 GiB (16 TB)                           │
│ → io2: 4 GiB - 16,384 GiB (64 TB for Block Express)         │
│ → st1/sc1: 125 GiB - 16,384 GiB                              │
│                                                                   │
│ IOPS: [3000]                                                   │
│ → gp3 baseline: 3,000 (free). Max: 16,000.                   │
│ → Extra IOPS cost: $0.005/IOPS/month above baseline           │
│ → io2: Up to 64,000 IOPS (ratio: 500 IOPS per GiB)          │
│ → ⚠️ Only SSD types have configurable IOPS                    │
│                                                                   │
│ Throughput (MiB/s): [125]                                     │
│ → gp3 baseline: 125 MB/s (free). Max: 1,000 MB/s.            │
│ → Extra throughput: $0.040/MB/s/month above baseline          │
│ → Only shown for gp3 and io2                                  │
│                                                                   │
│ Availability Zone: [us-east-1a ▼]                              │
│ → ⚠️ MUST match the AZ of the EC2 instance you'll attach to  │
│ → Volume and instance must be in the same AZ                  │
│ → Cannot be changed after creation (use snapshot to migrate)  │
│                                                                   │
│ ── Snapshot (optional) ──                                      │
│ Snapshot ID: [snap-0abc123...]                                │
│ → Create volume from an existing snapshot                     │
│ → Pre-populates the volume with snapshot data                 │
│ → Used for: restoring backups, cloning volumes, migration    │
│                                                                   │
│ ── Encryption ──                                                │
│ ☑ Encrypt this volume                                         │
│ KMS key: [aws/ebs (default) ▼]                                │
│   ├── aws/ebs: AWS-managed key (free, easy)                  │
│   ├── Custom CMK: Your managed key (more control)            │
│   └── Cross-account key: For shared snapshots                │
│ → Encryption is transparent to EC2 (no performance impact)   │
│ → ⚡ Always enable encryption for production                  │
│ → Once created, encryption cannot be changed                  │
│   (create encrypted snapshot → create new volume from it)    │
│                                                                   │
│ ── Tags ──                                                      │
│ Key: [Name]         Value: [data-vol-prod-01]                 │
│ Key: [environment]  Value: [production]                       │
│ Key: [team]         Value: [backend]                          │
│                                                                   │
│                    [Create volume]                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Attaching & Mounting Volumes

```
Console → EC2 → Volumes → Select volume → Actions → Attach volume

┌─────────────────────────────────────────────────────────────────┐
│           ATTACH VOLUME                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Instance: [i-0abc123def456 (web-prod-01) ▼]                   │
│ → Shows only instances in the same AZ as the volume           │
│                                                                   │
│ Device name: [/dev/sdf]                                        │
│ → Linux: /dev/sdf through /dev/sdp                            │
│ → Windows: xvdf through xvdp                                  │
│ → Note: Linux may rename (e.g., /dev/sdf → /dev/xvdf)       │
│ → Root volume is always /dev/sda1 or /dev/xvda                │
│                                                                   │
│                    [Attach volume]                              │
└─────────────────────────────────────────────────────────────────┘

After attaching — SSH into EC2 and mount:

# 1. List available disks
lsblk
# Output:
# NAME    SIZE TYPE
# xvda     8G  disk    ← root volume
# └─xvda1  8G  part
# xvdf   100G  disk    ← new volume (unmounted)

# 2. Check if volume has a file system
sudo file -s /dev/xvdf
# "data" means no file system → needs formatting
# "ext4" or "xfs" means already formatted

# 3. Create file system (ONLY if new, empty volume)
sudo mkfs -t xfs /dev/xvdf
# or: sudo mkfs -t ext4 /dev/xvdf

# 4. Create mount point
sudo mkdir /data

# 5. Mount the volume
sudo mount /dev/xvdf /data

# 6. Verify
df -h /data

# 7. Auto-mount on reboot (add to fstab)
# Get UUID
sudo blkid /dev/xvdf
# Add to /etc/fstab:
# UUID=xxxx-xxxx  /data  xfs  defaults,nofail  0  2

⚠️ NEVER format a volume that already has data (step 3)!
   Check with `file -s` first.
```

---

## Part 5: Snapshots

```
Console → EC2 → Snapshots → Create snapshot

┌─────────────────────────────────────────────────────────────────┐
│           CREATE SNAPSHOT                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Resource type: ● Volume ○ Instance                              │
│ → Volume: Snapshot a single EBS volume                         │
│ → Instance: Snapshot ALL volumes attached to an instance      │
│                                                                   │
│ Volume ID: [vol-0abc123...]  ▼                                 │
│ → Select the volume to snapshot                                │
│                                                                   │
│ Description: [Daily backup - web-prod-01 data volume]         │
│                                                                   │
│ ── Tags ──                                                      │
│ Key: [Name]         Value: [web-prod-01-data-2024-01-15]      │
│ Key: [backup-type]  Value: [daily]                             │
│                                                                   │
│                    [Create snapshot]                             │
│                                                                   │
│ ── How snapshots work ──                                       │
│ ├── First snapshot: Full copy of the volume → stored in S3   │
│ ├── Subsequent snapshots: Incremental (only changed blocks)  │
│ ├── Each snapshot is self-contained for restoration           │
│ ├── Snapshot is async — volume remains usable during snapshot │
│ ├── ⚡ Best practice: Snapshot from stopped instance           │
│ │   (ensures data consistency for databases)                  │
│ └── Cross-region: Copy snapshot to another region for DR      │
│                                                                   │
│ ── Create volume from snapshot ──                              │
│ Snapshots → Select → Actions → Create volume from snapshot   │
│ → Can change: AZ, volume type, size, encryption              │
│ → Use case: Migrate volume to different AZ                    │
│ → Use case: Upgrade from gp2 to gp3                          │
│ → Use case: Encrypt an unencrypted volume                     │
│                                                                   │
│ ── Snapshot lifecycle ──                                        │
│ EC2 → Lifecycle Manager → Create lifecycle policy             │
│   Description: [Daily EBS snapshots]                           │
│   Target: Volumes with tag [backup=true]                      │
│   Schedule: Every 24 hours at 03:00 UTC                       │
│   Retention: 7 snapshots (delete oldest)                      │
│   Cross-region copy: ☐ (enable for DR)                       │
│   → Automates snapshot creation and cleanup!                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Encryption

```
┌─────────────────────────────────────────────────────────────────────┐
│           EBS ENCRYPTION                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What's encrypted:                                                    │
│ ├── Data at rest on the volume                                    │
│ ├── Data in transit between instance and volume                  │
│ ├── All snapshots created from the volume                        │
│ └── All volumes created from encrypted snapshots                 │
│                                                                       │
│ Key points:                                                          │
│ ├── Uses AES-256 encryption                                      │
│ ├── Handled by EC2 host → no performance impact                 │
│ ├── Transparent to the OS and application                        │
│ ├── Uses AWS KMS keys (AWS-managed or customer-managed)         │
│ ├── Cannot encrypt an existing unencrypted volume directly       │
│ └── Default encryption: Enable at account level for all new vols │
│                                                                       │
│ Enable default encryption (account-wide):                           │
│ EC2 → Settings → EBS encryption → Manage                          │
│   ☑ Always encrypt new EBS volumes                                │
│   Default encryption key: [aws/ebs ▼]                              │
│   → ⚡ Enable this! All new volumes will be encrypted.             │
│                                                                       │
│ Encrypting an existing unencrypted volume:                          │
│ 1. Create snapshot of unencrypted volume                           │
│ 2. Copy snapshot → check "Encrypt this snapshot"                  │
│ 3. Create new volume from encrypted snapshot                      │
│ 4. Detach old volume, attach new encrypted volume                 │
│                                                                       │
│ ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │
│ │ Unencrypted  │ → │ Copy snap    │ → │ Encrypted    │            │
│ │ Volume       │   │ (encrypt)    │   │ Volume       │            │
│ │              │   │              │   │              │            │
│ │ snapshot ──→ │   │  encrypted   │   │ ← from snap  │            │
│ └──────────────┘   └──────────────┘   └──────────────┘           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Multi-Attach

```
┌─────────────────────────────────────────────────────────────────────┐
│           EBS MULTI-ATTACH                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Attach a single io2 volume to multiple EC2 instances         │
│ simultaneously (up to 16 Nitro-based instances).                    │
│                                                                       │
│ ┌────────────┐                                                      │
│ │ EC2 inst 1 │ ←──┐                                                │
│ ├────────────┤    │                                                │
│ │ EC2 inst 2 │ ←──┤── io2 Volume (Multi-Attach enabled)          │
│ ├────────────┤    │                                                │
│ │ EC2 inst 3 │ ←──┘                                                │
│ └────────────┘                                                      │
│                                                                       │
│ Requirements:                                                        │
│ ├── io2 or io1 volume types ONLY                                 │
│ ├── All instances must be in the same AZ                         │
│ ├── Nitro-based instances only                                    │
│ ├── Up to 16 instances simultaneously                            │
│ ├── Must use cluster-aware file system (GFS2, OCFS2)            │
│ └── ⚠️ NOT for general-purpose shared storage (use EFS instead)  │
│                                                                       │
│ Use cases:                                                           │
│ ├── Clustered databases (Oracle RAC)                              │
│ ├── Applications that manage concurrent write operations         │
│ └── High-availability scenarios                                   │
│                                                                       │
│ Enable: Check "Multi-Attach" when creating io2 volume             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Instance Store (Ephemeral Storage)

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTANCE STORE vs EBS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance Store:                                                      │
│ ├── Physically attached NVMe SSDs on the EC2 host                │
│ ├── Extremely high IOPS (millions of IOPS)                       │
│ ├── ⚠️ EPHEMERAL: Data LOST when instance stops/terminates       │
│ ├── Data survives reboot (but not stop/terminate)                │
│ ├── Cannot be detached or reattached                              │
│ ├── Size determined by instance type                              │
│ ├── Available on: i3, i4g, d3, c5d, m5d, r5d, etc.             │
│ └── FREE (included in instance price)                             │
│                                                                       │
│ ┌──────────────┬─────────────────┬──────────────────────────────┐ │
│ │              │ EBS              │ Instance Store               │ │
│ ├──────────────┼─────────────────┼──────────────────────────────┤ │
│ │ Persistence  │ Persists        │ Ephemeral (lost on stop)     │ │
│ │ Performance  │ Good-Great      │ Excellent (NVMe, millions    │ │
│ │              │                 │ of IOPS)                      │ │
│ │ Cost         │ Per GB/month    │ Free (with instance)          │ │
│ │ Detachable   │ Yes             │ No                            │ │
│ │ Snapshots    │ Yes             │ No                            │ │
│ │ Encryption   │ Yes (KMS)       │ Yes (hardware)                │ │
│ │ Size         │ 1 GB - 64 TB   │ Fixed by instance type        │ │
│ │ Best for     │ Databases, OS   │ Cache, temp files, scratch    │ │
│ └──────────────┴─────────────────┴──────────────────────────────┘ │
│                                                                       │
│ ⚡ Use Instance Store for:                                           │
│ ├── Temporary storage, scratch space                              │
│ ├── Cache (Redis, local cache)                                    │
│ ├── Buffers, temporary data processing                            │
│ └── Any data you can afford to lose                               │
│                                                                       │
│ ⚡ Use EBS for:                                                      │
│ ├── Databases (data must persist!)                                │
│ ├── OS/boot volumes                                                │
│ ├── Application data                                               │
│ └── Anything that needs to survive stop/restart                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Performance Optimization

```
┌─────────────────────────────────────────────────────────────────────┐
│           EBS PERFORMANCE OPTIMIZATION                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Factors affecting performance:                                      │
│ ├── Volume type (SSD vs HDD)                                     │
│ ├── Volume size (affects IOPS for gp2)                           │
│ ├── Provisioned IOPS/throughput (gp3, io2)                      │
│ ├── Instance type (EBS-optimized bandwidth)                      │
│ └── Pre-warming (for volumes from snapshots)                     │
│                                                                       │
│ EBS-Optimized instances:                                            │
│ ├── Dedicated bandwidth between EC2 and EBS                     │
│ ├── Most current-gen instances are EBS-optimized by default     │
│ ├── Bandwidth: 500 Mbps to 80 Gbps (depends on instance)       │
│ └── Check: Instance type → EBS bandwidth in docs                │
│                                                                       │
│ gp3 tuning:                                                         │
│ ├── Baseline: 3,000 IOPS + 125 MB/s (FREE with any size)       │
│ ├── Max: 16,000 IOPS + 1,000 MB/s                               │
│ ├── Can increase IOPS without increasing size                   │
│ ├── Can increase throughput without increasing IOPS              │
│ └── Modify: Volumes → Select → Actions → Modify volume          │
│                                                                       │
│ Pre-warming (initialization):                                       │
│ ├── Volumes from snapshots: blocks loaded lazily from S3        │
│ ├── First read of each block is slower (fetched from S3)        │
│ ├── Pre-warm by reading all blocks: sudo dd if=/dev/xvdf        │
│ │   of=/dev/null bs=1M status=progress                          │
│ ├── Or use Fast Snapshot Restore (FSR):                          │
│ │   Snapshots → Select → Manage Fast Snapshot Restore           │
│ │   → Enable for specific AZs                                   │
│ │   → $0.75/hour per snapshot-AZ pair (expensive!)              │
│ └── ⚡ Use FSR for production databases restored from snapshots  │
│                                                                       │
│ RAID configurations:                                                 │
│ ├── RAID 0: Stripe across volumes (more IOPS + throughput)     │
│ ├── RAID 1: Mirror for redundancy (EBS already replicates)     │
│ └── ⚡ RAID 0 is common when you need >16K IOPS without io2    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9.5: Elastic Volumes — Modify Without Downtime

**Elastic Volumes** let you change the size, type, or IOPS of an attached EBS volume **without stopping the instance or detaching the volume**. This is a game-changer for production.

### What Can You Modify?

| Change | Supported? | Notes |
|--------|-----------|-------|
| Increase size | ✅ Yes | Cannot decrease size (ever) |
| Change type (gp3 → io2) | ✅ Yes | Some types require reboot |
| Change IOPS | ✅ Yes | gp3/io1/io2 only |
| Change throughput | ✅ Yes | gp3 only |
| Decrease size | ❌ No | Must create new smaller volume + copy data |

### Console Walkthrough

```
Console → EC2 → Volumes → Select volume → Actions → Modify volume

  ┌─────────────────────────────────────────────────────────┐
  │ Modify Volume                                            │
  │                                                          │
  │ Volume type:     gp3           ← can change              │
  │ Size (GiB):      100 → 200    ← can only increase        │
  │ IOPS:            3000 → 6000  ← gp3/io1/io2 only        │
  │ Throughput:      125 → 250    ← gp3 only (MiB/s)        │
  │                                                          │
  │ [Modify]                                                 │
  └─────────────────────────────────────────────────────────┘
```

### After Modification

```
1. Volume state changes to "optimizing" (may take hours for large changes)
2. For SIZE increases, you must also extend the file system:

   Linux:
   sudo growpart /dev/xvda 1          # Extend the partition
   sudo resize2fs /dev/xvda1          # Extend ext4 filesystem
   # OR: sudo xfs_growfs /            # If using XFS

   Windows:
   Disk Management → Right-click volume → Extend Volume
```

> ⚠️ **Cooldown:** After modifying a volume, you must wait **6 hours** before making another modification to the same volume.

### CLI Example

```bash
aws ec2 modify-volume \
  --volume-id vol-0123456789abcdef0 \
  --size 200 \
  --volume-type gp3 \
  --iops 6000 \
  --throughput 250
```

### Delete on Termination — Don't Forget!

By default, the **root volume** is deleted when the EC2 instance is terminated, but **additional volumes are NOT**. This means you could be paying for orphaned EBS volumes.

```
Check: Console → EC2 → Instances → Storage tab
  Root volume:       Delete on termination = Yes (default)
  Additional volume: Delete on termination = No  ← ⚠️ Will be orphaned!

Fix: Set DeleteOnTermination = true when attaching, or clean up regularly.
```

---

## Part 10: Terraform & CLI Examples

```hcl
# EBS Volume
resource "aws_ebs_volume" "data" {
  availability_zone = "us-east-1a"
  size              = 100
  type              = "gp3"
  iops              = 3000
  throughput        = 125
  encrypted         = true

  tags = {
    Name        = "data-vol-prod-01"
    Environment = "prod"
  }
}

# Attach to EC2
resource "aws_volume_attachment" "data" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.data.id
  instance_id = aws_instance.web.id
}

# Snapshot lifecycle policy
resource "aws_dlm_lifecycle_policy" "daily_snapshots" {
  description        = "Daily EBS snapshots"
  execution_role_arn = aws_iam_role.dlm.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]
    target_tags = { backup = "true" }

    schedule {
      name = "Daily snapshots"
      create_rule { interval = 24; interval_unit = "HOURS"; times = ["03:00"] }
      retain_rule { count = 7 }
      copy_tags = true
    }
  }
}
```

```bash
# Create volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=data-vol}]'

# Attach volume to instance
aws ec2 attach-volume \
  --volume-id vol-0abc123 \
  --instance-id i-0def456 \
  --device /dev/sdf

# Create snapshot
aws ec2 create-snapshot \
  --volume-id vol-0abc123 \
  --description "Daily backup" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=daily-backup}]'

# Copy snapshot to another region
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0abc123 \
  --destination-region eu-west-1 \
  --encrypted

# Modify volume (e.g., increase IOPS)
aws ec2 modify-volume --volume-id vol-0abc123 --iops 6000 --throughput 500

# List volumes
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType}"
```

---

## Part 11: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD EBS PATTERNS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Production Database                                      │
│ ├── Volume: io2, 500 GB, 20,000 IOPS                            │
│ ├── Encryption: Customer-managed KMS key                         │
│ ├── Snapshots: Every 6 hours via DLM, retain 14 days            │
│ ├── Cross-region snapshot copy for DR                            │
│ └── Fast Snapshot Restore enabled in DR region                   │
│                                                                       │
│ Pattern 2: Web Server                                               │
│ ├── Root volume: gp3, 20 GB                                     │
│ ├── Data volume: gp3, 50 GB (application data)                  │
│ ├── Daily snapshots, retain 7 days                               │
│ └── Delete on termination: root=yes, data=no                    │
│                                                                       │
│ Pattern 3: Big Data Processing                                      │
│ ├── Instance store (i3 instance) for temp/scratch                │
│ ├── st1 volumes for large sequential reads                       │
│ ├── Results written to S3                                        │
│ └── Volumes deleted after processing complete                    │
│                                                                       │
│ Pattern 4: Cost Optimization                                        │
│ ├── Migrate gp2 → gp3 (20% savings, same or better perf)      │
│ ├── Right-size volumes (check CloudWatch VolumeReadOps)         │
│ ├── Delete unattached volumes (common waste)                    │
│ ├── Delete old snapshots (use lifecycle policies)               │
│ └── Use st1 for infrequently accessed large datasets            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
EBS Quick Reference:
├── Volume types: gp3 (default), gp2, io2, st1, sc1
├── Max size: 16 TB (64 TB for io2 Block Express)
├── Max IOPS: 16K (gp3), 64K (io2), 256K (io2 Block Express)
├── Scope: Single AZ (use snapshots to migrate)
├── Snapshots: Incremental, stored in S3
├── Encryption: AES-256 via KMS, no performance impact
├── Multi-Attach: io2 only, up to 16 instances
├── Instance Store: Ephemeral, ultra-high IOPS, data lost on stop
├── DLM: Automate snapshot lifecycle
└── ⚡ Always use gp3 over gp2, always enable encryption
```

---

## What's Next?

In **Chapter 23: EFS - Elastic File System**, we'll cover shared file storage that can be mounted by multiple EC2 instances simultaneously across availability zones.
