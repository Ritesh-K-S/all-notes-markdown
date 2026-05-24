# Chapter 23: Filestore

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Filestore Fundamentals](#part-1-filestore-fundamentals)
- [Part 2: Service Tiers](#part-2-service-tiers)
- [Part 3: Creating a Filestore Instance](#part-3-creating-a-filestore-instance)
- [Part 4: Mounting & Accessing File Shares](#part-4-mounting--accessing-file-shares)
- [Part 5: Performance & Scaling](#part-5-performance--scaling)
- [Part 6: Backups](#part-6-backups)
- [Part 7: Snapshots](#part-7-snapshots)
- [Part 8: Networking & Access Control](#part-8-networking--access-control)
- [Part 9: Filestore Multishares for GKE](#part-9-filestore-multishares-for-gke)
- [Part 10: Monitoring & Troubleshooting](#part-10-monitoring--troubleshooting)
- [Part 11: Terraform & CLI](#part-11-terraform--cli)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Filestore is Google Cloud's fully managed NFS file server. It provides a shared filesystem that multiple VMs, GKE pods, or on-premises clients can mount simultaneously. Unlike Persistent Disk (block storage, single-writer), Filestore supports multi-reader/multi-writer access — ideal for shared data, CMS platforms, media processing, and HPC workloads.

```
What you'll learn:
├── Filestore Fundamentals
│   ├── Managed NFS (Network File System)
│   ├── When to use Filestore vs PD vs GCS
│   ├── Shared read/write across multiple clients
│   └── File system semantics (POSIX)
├── Service Tiers
│   ├── Basic HDD — cheapest, dev/test
│   ├── Basic SSD — general purpose
│   ├── Zonal — high performance, SSD
│   ├── Regional — HA, cross-zone replication
│   └── Enterprise — mission-critical, multi-zone HA
├── Creating Instances
│   ├── Console walkthrough (all fields)
│   ├── Capacity provisioning
│   └── Network configuration
├── Mounting & Access
│   ├── Linux NFS mount
│   ├── GKE Persistent Volume
│   └── Multiple VMs sharing same mount
├── Performance & Scaling
│   ├── IOPS and throughput per tier
│   ├── Scaling capacity online
│   └── Performance tuning
├── Backups & Snapshots
├── Networking & Access Control
├── Filestore Multishares for GKE
├── Monitoring & Troubleshooting
├── Terraform & CLI
└── Real-world patterns
```

---

## Part 1: Filestore Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS FILESTORE?                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Filestore = Managed NFS file server on Google Cloud.               │
│                                                                       │
│ NFS (Network File System):                                          │
│ ├── Standard protocol for sharing files over a network            │
│ ├── Looks like a local directory — use normal file operations    │
│ ├── POSIX-compliant (permissions, symlinks, locking, etc.)       │
│ ├── Multiple clients can read AND write simultaneously           │
│ └── NFSv3 protocol (NFSv4.1 on Enterprise tier)                 │
│                                                                       │
│ ┌──────────┐     ┌──────────┐     ┌──────────┐                    │
│ │  VM #1   │     │  VM #2   │     │  VM #3   │                    │
│ │ (writer) │     │ (writer) │     │ (reader) │                    │
│ └────┬─────┘     └────┬─────┘     └────┬─────┘                    │
│      │                │                │                           │
│      └────────────────┼────────────────┘                           │
│                       │ NFS mount                                  │
│              ┌────────▼─────────┐                                  │
│              │   FILESTORE      │                                  │
│              │   Instance       │                                  │
│              │  ┌─────────────┐ │                                  │
│              │  │ File Share  │ │                                  │
│              │  │ /vol1       │ │                                  │
│              │  │ 1-100 TB    │ │                                  │
│              │  └─────────────┘ │                                  │
│              └──────────────────┘                                  │
│                                                                       │
│ Key properties:                                                     │
│ ├── FULLY MANAGED: No OS patching, no NFS server maintenance    │
│ ├── SHARED: Multiple VMs/pods mount the same filesystem          │
│ ├── READ + WRITE: All clients can read and write (not read-only)│
│ ├── POSIX: Full file system semantics (unlike object storage)   │
│ ├── LOW LATENCY: Sub-millisecond for SSD tiers                  │
│ ├── CONSISTENT: Strong consistency (read-after-write)           │
│ └── CAPACITY: Scale from 1 TB to 100 TB per instance            │
│                                                                       │
│ AWS equivalent: Amazon EFS (Elastic File System)                   │
│ Azure equivalent: Azure Files (SMB/NFS) / Azure NetApp Files     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### When to Use Filestore vs Persistent Disk vs Cloud Storage

```
┌──────────────────────┬─────────────────┬─────────────────┬───────────────────┐
│                      │ Persistent Disk │ Filestore       │ Cloud Storage     │
│                      │ (Block)         │ (File/NFS)      │ (Object)          │
├──────────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Protocol             │ Block device    │ NFS             │ HTTP/REST API     │
│ Access pattern       │ Single VM (RW)  │ Multiple VMs    │ Multiple clients  │
│                      │ Multi VM (RO)   │ (RW)            │ (RW via API)      │
│ File system          │ ✅ You format   │ ✅ Provided     │ ❌ No filesystem  │
│ POSIX compatible     │ ✅ (you manage) │ ✅ Built-in     │ ❌ No             │
│ Latency              │ Sub-ms          │ Sub-ms (SSD)    │ ~10-100ms         │
│ Max size             │ 64 TB           │ 100 TB          │ Unlimited         │
│ Multi-writer         │ ❌              │ ✅              │ ✅ (via API)      │
│ Cost per TB          │ ~$40-170        │ ~$200-600       │ ~$20-26           │
│                      │                 │                 │                   │
│ Best for             │ Databases,      │ Shared content, │ Static files,     │
│                      │ single-VM       │ CMS, media,     │ backups, data     │
│                      │ workloads       │ HPC, GKE        │ lake, archival    │
└──────────────────────┴─────────────────┴─────────────────┴───────────────────┘

⚡ Decision:
├── Need block device for database?          → Persistent Disk
├── Need shared filesystem for multiple VMs? → Filestore
├── Need unlimited cheap storage via API?    → Cloud Storage
├── Need shared storage for GKE pods?        → Filestore (or GCS FUSE)
└── Need file locking / POSIX semantics?     → Filestore
```

---

## Part 2: Service Tiers

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              FILESTORE SERVICE TIERS                                            │
├────────────────┬─────────────┬─────────────┬─────────────┬─────────────┬────────────────────┤
│ Property       │ Basic HDD   │ Basic SSD   │ Zonal       │ Regional    │ Enterprise         │
├────────────────┼─────────────┼─────────────┼─────────────┼─────────────┼────────────────────┤
│ Storage        │ HDD         │ SSD         │ SSD         │ SSD         │ SSD                │
│                │             │             │             │             │                    │
│ Min capacity   │ 1 TB        │ 2.5 TB      │ 1 TB        │ 1 TB        │ 1 TB               │
│ Max capacity   │ 63.9 TB     │ 63.9 TB     │ 100 TB      │ 100 TB      │ 10 TB              │
│                │             │             │             │             │                    │
│ Max read IOPS  │ 600/TB      │ 60,000      │ 500,000     │ 500,000     │ 120,000            │
│ Max write IOPS │ 1,000/TB    │ 25,000      │ 120,000     │ 120,000     │ 40,000             │
│                │             │             │             │             │                    │
│ Read MB/s      │ 100/TB      │ 1,200       │ 26,000      │ 26,000      │ 1,200              │
│ Write MB/s     │ 100/TB      │ 350         │ 4,800       │ 4,800       │ 350                │
│                │             │             │             │             │                    │
│ Availability   │ Zonal       │ Zonal       │ Zonal       │ Regional ✅ │ Regional ✅         │
│                │             │             │             │ (2 zones)   │ (2 zones)          │
│                │             │             │             │             │                    │
│ NFS version    │ NFSv3       │ NFSv3       │ NFSv3       │ NFSv3       │ NFSv3 + NFSv4.1   │
│                │             │             │             │             │                    │
│ Snapshots      │ ❌          │ ❌          │ ✅          │ ✅          │ ✅                  │
│ Backups        │ ✅          │ ✅          │ ✅          │ ✅          │ ✅                  │
│                │             │             │             │             │                    │
│ Online resize  │ ❌ (scale   │ ❌ (scale   │ ✅ Up &     │ ✅ Up &     │ ✅ Up & down       │
│                │ up only)    │ up only)    │ down        │ down        │                    │
│                │             │             │             │             │                    │
│ Price/TB/month │ ~$200       │ ~$600       │ ~$350       │ ~$700       │ ~$900              │
│ (approx)       │ (cheapest)  │             │(best perf/$)│             │ (most expensive)   │
│                │             │             │             │             │                    │
│ Use case       │ Dev/test,   │ General     │ High-perf   │ HA prod     │ Mission-critical,  │
│                │ low-traffic │ purpose     │ workloads,  │ workloads,  │ enterprise apps    │
│                │ file shares │ prod        │ media, HPC  │ HA required │                    │
├────────────────┴─────────────┴─────────────┴─────────────┴─────────────┴────────────────────┤
│                                                                                               │
│ ⚡ CHOOSING A TIER:                                                                            │
│ ├── Budget dev/test, low IOPS needs          → Basic HDD                                     │
│ ├── Standard production, moderate IOPS       → Basic SSD                                     │
│ ├── High performance, need scale up & down   → Zonal (best perf/$)                           │
│ ├── HA required (multi-zone)                 → Regional                                      │
│ └── Enterprise compliance, NFSv4.1, mission  → Enterprise                                   │
│     critical (but lower max capacity)                                                        │
│                                                                                               │
│ ⚡ Zonal tier is the sweet spot for most production workloads.                                 │
│   Regional if you need zone-level HA.                                                        │
│                                                                                               │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud File Storage Comparison

```
┌──────────────────────────┬──────────────────┬──────────────────────┐
│ GCP Filestore            │ AWS EFS          │ Azure Files          │
├──────────────────────────┼──────────────────┼──────────────────────┤
│ Basic HDD                │ EFS Infrequent   │ Standard (SMB/NFS)   │
│ Basic SSD                │ EFS Standard     │ Premium (SMB)        │
│ Zonal                    │ EFS (regional)   │ Premium NFS          │
│ Regional                 │ EFS (regional)   │ Premium ZRS          │
│ Enterprise               │ EFS + advanced   │ Azure NetApp Files   │
│                          │                  │ Ultra tier            │
├──────────────────────────┼──────────────────┼──────────────────────┤
│ Protocol: NFS            │ NFS              │ SMB + NFS            │
│ POSIX: ✅                │ POSIX: ✅        │ NFS: ✅ SMB: partial │
│ Max size: 100 TB         │ Unlimited (PB)   │ 100 TiB              │
│ Pricing: Per provisioned │ Per used GB      │ Per provisioned      │
│ capacity                 │ (elastic) ✅      │ capacity             │
└──────────────────────────┴──────────────────┴──────────────────────┘

⚡ AWS EFS charges per used GB (elastic) — you don't pre-provision.
   Filestore charges for PROVISIONED capacity — you pay for the
   full 1 TB even if you use 100 GB. Plan capacity carefully!
```

---

## Part 3: Creating a Filestore Instance

```
Console → Filestore → Instances → Create Instance

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE FILESTORE INSTANCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance ID: [shared-data-fs]                                      │
│ Description: [Shared file system for web tier]                     │
│                                                                       │
│ Instance type (tier):                                               │
│   ○ Basic                                                           │
│     └── Storage type: ○ HDD  ○ SSD                                │
│   ○ Zonal                                                           │
│   ○ Regional                                                        │
│   ○ Enterprise                                                      │
│                                                                       │
│ Capacity:                                                           │
│   Allocated capacity: [2.5] TB                                     │
│   ⚡ Basic HDD min = 1 TB, Basic SSD min = 2.5 TB                 │
│   ⚡ Zonal/Regional/Enterprise min = 1 TB                          │
│   ⚡ You pay for ALLOCATED capacity, not used!                      │
│                                                                       │
│ Region: [asia-south1]                                              │
│ Zone: [asia-south1-a]  (not for Regional/Enterprise)              │
│                                                                       │
│ File share:                                                         │
│   File share name: [vol1]                                          │
│   ⚡ This is the NFS export name — clients mount this             │
│   ⚡ Basic/Zonal: 1 file share per instance                       │
│   ⚡ Enterprise: Up to 10 file shares per instance                │
│                                                                       │
│ VPC network:                                                        │
│   Network: [default] or [custom-vpc]                               │
│   ⚡ Filestore uses a PRIVATE IP in your VPC                       │
│   ⚡ The instance gets an IP like 10.0.1.2                         │
│   ⚡ VMs in the same VPC can mount via this IP                     │
│                                                                       │
│   IP range:                                                         │
│   ○ Use an automatically allocated IP range (recommended)         │
│   ○ Use a reserved IP range                                       │
│   ⚡ Filestore reserves a /29 CIDR block in your VPC              │
│                                                                       │
│ Access control:                                                     │
│   (Who can mount this file share — IP-based)                      │
│   ├── IP range: [10.0.0.0/8]  (all VMs in VPC)                  │
│   ├── Access mode:                                                │
│   │   ○ Admin (read-write + root squash disabled)                │
│   │   ○ Read-Write                                                │
│   │   ○ Read-Only                                                 │
│   └── Root squash: ☐ Enable (map root to nobody)                 │
│                                                                       │
│ Encryption:                                                         │
│   ○ Google-managed encryption key (default)                       │
│   ○ Customer-managed encryption key (CMEK)                        │
│     → KMS key: [projects/proj/locations/loc/keyRings/kr/keys/k]  │
│                                                                       │
│ Labels:                                                              │
│   environment: prod                                                │
│   team: web                                                         │
│                                                                       │
│ [CREATE]                                                             │
│                                                                       │
│ ⚡ Creation takes 5-15 minutes.                                     │
│ ⚡ After creation you'll see the instance IP address — note it!    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Mounting & Accessing File Shares

### Mounting on Linux VMs

```
┌─────────────────────────────────────────────────────────────────────┐
│           MOUNT FILESTORE ON LINUX VM                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Prerequisites:                                                      │
│ ├── VM must be in the SAME VPC as the Filestore instance         │
│ ├── NFS client must be installed on the VM                       │
│ └── Firestore instance IP (e.g., 10.0.1.2)                      │
│                                                                       │
│ Step 1: Install NFS client                                         │
│ # Debian/Ubuntu:                                                   │
│ sudo apt-get update                                                │
│ sudo apt-get install -y nfs-common                                 │
│                                                                       │
│ # RHEL/CentOS:                                                     │
│ sudo yum install -y nfs-utils                                      │
│                                                                       │
│ Step 2: Create mount point                                         │
│ sudo mkdir -p /mnt/shared                                          │
│                                                                       │
│ Step 3: Mount the file share                                       │
│ sudo mount 10.0.1.2:/vol1 /mnt/shared                             │
│ #         ^^^^^^^^  ^^^^   ^^^^^^^^^^^                             │
│ #         FS IP    share   local mount                             │
│ #                  name    point                                   │
│                                                                       │
│ Step 4: Verify                                                     │
│ df -h /mnt/shared                                                  │
│ # Filesystem       Size  Used Avail Use% Mounted on               │
│ # 10.0.1.2:/vol1   2.5T  256K  2.5T   1% /mnt/shared             │
│                                                                       │
│ Step 5: Auto-mount on reboot (add to /etc/fstab)                  │
│ echo "10.0.1.2:/vol1 /mnt/shared nfs defaults,_netdev 0 0" \    │
│   | sudo tee -a /etc/fstab                                        │
│                                                                       │
│ ⚡ Use _netdev option — tells OS to wait for network before mount │
│ ⚡ Same file share is mounted on ALL VMs — they see the same data│
│                                                                       │
│ Mount options for performance:                                     │
│ sudo mount -o rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2│
│   10.0.1.2:/vol1 /mnt/shared                                      │
│                                                                       │
│ ├── rsize/wsize=1048576: 1 MB read/write buffer (max throughput)│
│ ├── hard: Retry indefinitely on server failure                   │
│ ├── timeo=600: 60-second timeout before retry                    │
│ └── retrans=2: Retry 2 times before reporting error             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Accessing from GKE (Kubernetes)

```
┌─────────────────────────────────────────────────────────────────────┐
│           FILESTORE WITH GKE                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two methods to use Filestore in GKE:                               │
│                                                                       │
│ Method 1: Manual PV/PVC (pre-provisioned Filestore)               │
│ ──────────────────────────────────────────────────                  │
│                                                                       │
│ # PersistentVolume (points to existing Filestore)                 │
│ apiVersion: v1                                                     │
│ kind: PersistentVolume                                              │
│ metadata:                                                           │
│   name: filestore-pv                                               │
│ spec:                                                               │
│   capacity:                                                         │
│     storage: 2.5Ti                                                 │
│   accessModes:                                                      │
│     - ReadWriteMany          # ← Multiple pods can write!        │
│   nfs:                                                              │
│     server: 10.0.1.2         # Filestore instance IP              │
│     path: /vol1              # File share name                    │
│ ---                                                                 │
│ # PersistentVolumeClaim                                            │
│ apiVersion: v1                                                     │
│ kind: PersistentVolumeClaim                                        │
│ metadata:                                                           │
│   name: filestore-pvc                                              │
│ spec:                                                               │
│   accessModes:                                                      │
│     - ReadWriteMany                                                │
│   storageClassName: ""       # Empty = bind to specific PV        │
│   resources:                                                        │
│     requests:                                                       │
│       storage: 2.5Ti                                               │
│                                                                       │
│ Method 2: Filestore CSI Driver (dynamic provisioning) ✅           │
│ ──────────────────────────────────────────────────────              │
│                                                                       │
│ ⚡ GKE has a built-in Filestore CSI driver — enable it!            │
│                                                                       │
│ # Enable CSI driver on GKE cluster                                │
│ gcloud container clusters update my-cluster \                     │
│   --update-addons=GcpFilestoreCsiDriver=ENABLED \                │
│   --zone=asia-south1-a                                            │
│                                                                       │
│ # StorageClass for dynamic provisioning                           │
│ apiVersion: storage.k8s.io/v1                                     │
│ kind: StorageClass                                                  │
│ metadata:                                                           │
│   name: filestore-sc                                               │
│ provisioner: filestore.csi.storage.gke.io                         │
│ parameters:                                                         │
│   tier: standard             # basic-hdd, basic-ssd, zonal, etc. │
│   network: default                                                 │
│ volumeBindingMode: Immediate                                      │
│ allowVolumeExpansion: true                                         │
│ ---                                                                 │
│ # PVC — Filestore instance created automatically!                 │
│ apiVersion: v1                                                     │
│ kind: PersistentVolumeClaim                                        │
│ metadata:                                                           │
│   name: my-shared-data                                             │
│ spec:                                                               │
│   accessModes:                                                      │
│     - ReadWriteMany                                                │
│   storageClassName: filestore-sc                                   │
│   resources:                                                        │
│     requests:                                                       │
│       storage: 1Ti                                                 │
│                                                                       │
│ ⚡ Dynamic provisioning creates a NEW Filestore instance for each │
│   PVC. This can get expensive — use Multishares (Part 9) to     │
│   share one instance across multiple PVCs.                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Performance & Scaling

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERFORMANCE CHARACTERISTICS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Performance scales with TIER and CAPACITY:                          │
│                                                                       │
│ Basic HDD:                                                          │
│ ├── Read throughput: 100 MB/s per TB                              │
│ ├── Write throughput: 100 MB/s per TB                             │
│ ├── Read IOPS: 600 per TB                                        │
│ └── 10 TB instance = 1,000 MB/s read, 6,000 read IOPS          │
│                                                                       │
│ Basic SSD:                                                          │
│ ├── Read throughput: 1,200 MB/s (flat, not per TB)              │
│ ├── Write throughput: 350 MB/s (flat)                            │
│ ├── Read IOPS: 60,000 (flat)                                    │
│ └── Same performance whether 2.5 TB or 60 TB                   │
│                                                                       │
│ Zonal:                                                              │
│ ├── Read IOPS: Up to 500,000                                    │
│ ├── Write IOPS: Up to 120,000                                   │
│ ├── Read throughput: Up to 26,000 MB/s                          │
│ ├── Write throughput: Up to 4,800 MB/s                          │
│ └── ⚡ Performance scales with capacity up to these maximums     │
│                                                                       │
│ ⚡ Basic tiers: FIXED performance (doesn't scale with capacity)   │
│ ⚡ Zonal/Regional: SCALES with capacity (bigger = faster)         │
│ ⚡ This is the opposite of EFS where performance is automatic.    │
│                                                                       │
│ Client-side factors:                                                │
│ ├── VM machine type affects max NFS throughput                   │
│ ├── Small VMs (e2-micro) bottleneck NFS performance             │
│ ├── Number of concurrent clients affects aggregate throughput   │
│ └── Network bandwidth limit per VM applies                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Scaling Capacity

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCALING FILESTORE CAPACITY                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Basic tiers (HDD/SSD):                                              │
│ ├── Scale UP only (cannot decrease)                              │
│ ├── Online — no downtime during resize                          │
│ ├── Increase in 1 GB increments                                 │
│ └── Takes a few minutes                                          │
│                                                                       │
│ Zonal / Regional / Enterprise:                                     │
│ ├── Scale UP and DOWN ✅                                          │
│ ├── Online — no downtime                                         │
│ ├── Can decrease to min tier size (1 TB)                         │
│ ├── ⚡ Scale down to save costs when demand drops!               │
│ └── Cannot go below used capacity                               │
│                                                                       │
│ Console: Filestore → Instance → Edit → Change capacity → Save    │
│                                                                       │
│ CLI:                                                                │
│ gcloud filestore instances update shared-data-fs \                │
│   --file-share=name=vol1,capacity=5TB \                           │
│   --zone=asia-south1-a                                            │
│                                                                       │
│ ⚡ Unlike AWS EFS which auto-scales, Filestore requires manual     │
│   capacity management. Set up alerts on disk usage!              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Backups

```
┌─────────────────────────────────────────────────────────────────────┐
│           FILESTORE BACKUPS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Full copy of file share data, stored independently.          │
│       Survives instance deletion. Can restore to a new instance.  │
│                                                                       │
│ Key properties:                                                     │
│ ├── FULL COPY: Not incremental (each backup is complete)         │
│ ├── INDEPENDENT: Stored separately from the instance             │
│ │   (deleting instance does NOT delete backups)                  │
│ ├── REGIONAL: Stored in the same region as the instance         │
│ ├── RESTORE: Create a new Filestore instance from backup        │
│ │   (can be different tier, different zone, different size)      │
│ ├── CONSISTENT: Crash-consistent snapshot of file system        │
│ └── ALL TIERS: Available on all tiers                            │
│                                                                       │
│ Backup vs Snapshot:                                                 │
│ ┌──────────────┬───────────────┬───────────────────────┐          │
│ │              │ Backup        │ Snapshot              │          │
│ ├──────────────┼───────────────┼───────────────────────┤          │
│ │ Storage      │ Independent   │ On instance (shared   │          │
│ │              │ (separate)    │ capacity)             │          │
│ │ Survives     │ ✅ Yes        │ ❌ Lost with instance │          │
│ │ instance del │               │                       │          │
│ │ Copy type    │ Full          │ Incremental           │          │
│ │ Restore to   │ New instance  │ Same instance         │          │
│ │ Cross-region │ ❌ Same region│ ❌ Same instance      │          │
│ │ Speed        │ Slower        │ Fast (instant)        │          │
│ │ Cost         │ Per used GB   │ Uses instance capacity│          │
│ │ Tiers        │ All           │ Zonal/Regional/Ent.   │          │
│ └──────────────┴───────────────┴───────────────────────┘          │
│                                                                       │
│ ⚡ Backups = disaster recovery (survives instance failure)         │
│ ⚡ Snapshots = quick restore / rollback (faster, cheaper)          │
│                                                                       │
│ Console: Filestore → Instance → Backups → Create Backup          │
│                                                                       │
│ CLI:                                                                │
│ # Create backup                                                   │
│ gcloud filestore backups create daily-backup-20260517 \           │
│   --instance=shared-data-fs \                                     │
│   --file-share=vol1 \                                             │
│   --instance-zone=asia-south1-a \                                 │
│   --region=asia-south1                                            │
│                                                                       │
│ # List backups                                                     │
│ gcloud filestore backups list --region=asia-south1                │
│                                                                       │
│ # Restore: create new instance from backup                        │
│ gcloud filestore instances restore shared-data-restored \         │
│   --file-share=name=vol1,source-backup=daily-backup-20260517,    │
│     source-backup-region=asia-south1                              │
│                                                                       │
│ # Delete backup                                                   │
│ gcloud filestore backups delete daily-backup-20260517 \           │
│   --region=asia-south1                                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Snapshots

```
┌─────────────────────────────────────────────────────────────────────┐
│           FILESTORE SNAPSHOTS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Point-in-time, incremental capture of file share state.      │
│       Stored ON the instance (uses instance capacity).            │
│       Fast to create and restore.                                 │
│                                                                       │
│ Available on: Zonal, Regional, Enterprise tiers ONLY.             │
│ ❌ NOT available on Basic HDD / Basic SSD.                         │
│                                                                       │
│ Properties:                                                         │
│ ├── INCREMENTAL: Only changed blocks since last snapshot         │
│ ├── INSTANT: Near-instant creation                               │
│ ├── ON-INSTANCE: Uses capacity of the Filestore instance        │
│ │   ⚡ If snapshot data fills the instance → no space for writes│
│ │   Plan capacity: instance capacity > data + snapshots         │
│ ├── MAX: Up to 240 snapshots per instance                       │
│ ├── RESTORE: Revert file share to snapshot point-in-time        │
│ │   (in-place restore, replaces current data!)                  │
│ └── ⚠️ Deleted with instance — NOT for disaster recovery!        │
│     Use BACKUPS for DR.                                          │
│                                                                       │
│ Use cases:                                                          │
│ ├── Quick rollback before risky changes                         │
│ ├── Test data manipulation (snapshot → test → revert)           │
│ ├── Point-in-time data access (read from .snapshot directory)   │
│ └── Scheduled snapshots for continuous protection               │
│                                                                       │
│ Accessing snapshots (read-only, without full restore):            │
│ # Snapshots are accessible via a hidden .snapshot directory!     │
│ ls /mnt/shared/.snapshot/                                          │
│ # Lists all snapshot names (timestamped)                          │
│ ls /mnt/shared/.snapshot/snap-20260517/                           │
│ # Browse files as they were at snapshot time                     │
│ # ⚡ You can copy individual files from a snapshot!               │
│ cp /mnt/shared/.snapshot/snap-20260517/config.yaml /mnt/shared/  │
│                                                                       │
│ CLI:                                                                │
│ # Create snapshot                                                  │
│ gcloud filestore instances snapshots create snap-20260517 \       │
│   --instance=shared-data-fs \                                     │
│   --region=asia-south1    # or --zone for zonal                  │
│                                                                       │
│ # List snapshots                                                   │
│ gcloud filestore instances snapshots list \                       │
│   --instance=shared-data-fs \                                     │
│   --region=asia-south1                                            │
│                                                                       │
│ # Revert to snapshot (⚠️ replaces ALL current data!)              │
│ gcloud filestore instances revert shared-data-fs \                │
│   --target-snapshot=snap-20260517 \                               │
│   --region=asia-south1                                            │
│                                                                       │
│ # Delete snapshot                                                  │
│ gcloud filestore instances snapshots delete snap-20260517 \       │
│   --instance=shared-data-fs \                                     │
│   --region=asia-south1                                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Networking & Access Control

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING & ACCESS CONTROL                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Filestore uses PRIVATE IPs in your VPC. No public IPs.            │
│                                                                       │
│ Network architecture:                                               │
│ ┌─────────────────────────────────────────────────────┐            │
│ │  Your VPC (10.0.0.0/8)                               │            │
│ │                                                       │            │
│ │  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │            │
│ │  │ VM 10.0.1.5│  │ VM 10.0.1.6│  │ Filestore    │  │            │
│ │  │ NFS client │  │ NFS client │  │ 10.0.1.2     │  │            │
│ │  └──────┬─────┘  └──────┬─────┘  │ /vol1        │  │            │
│ │         └──────┬─────────┘        └──────▲───────┘  │            │
│ │                └─────────────────────────┘           │            │
│ │                    NFS over TCP port 2049           │            │
│ └─────────────────────────────────────────────────────┘            │
│                                                                       │
│ Access control layers:                                              │
│                                                                       │
│ 1. VPC NETWORK:                                                     │
│    ├── Filestore instance lives in a specific VPC               │
│    ├── Only VMs in the SAME VPC (or peered VPCs) can access    │
│    └── Cross-VPC: Use VPC Peering or Shared VPC                 │
│                                                                       │
│ 2. IP-BASED ACCESS RULES (on Filestore instance):                  │
│    ├── Define which IP ranges can mount the file share          │
│    ├── Per IP range, set access mode:                           │
│    │   ├── admin: read-write + root access                     │
│    │   ├── read-write: normal read-write                       │
│    │   └── read-only: read only                                 │
│    ├── Root squash: Map root (UID 0) to nobody (security)      │
│    │   ⚡ Enable for shared environments — prevents clients     │
│    │     from creating files as root                            │
│    └── Example:                                                  │
│        ├── 10.0.1.0/24: read-write (app servers)               │
│        ├── 10.0.2.0/24: read-only (reporting servers)          │
│        └── 10.0.0.5/32: admin (management VM)                  │
│                                                                       │
│ 3. FIREWALL RULES:                                                  │
│    ├── Allow TCP port 2049 (NFS)                                 │
│    ├── Allow TCP port 111 (portmapper/rpcbind)                  │
│    ├── Allow TCP/UDP ports 635, 4045 (mountd, lockd)           │
│    └── Default VPC allows internal traffic — usually no changes│
│        But custom VPCs may need explicit rules.                  │
│                                                                       │
│ 4. POSIX PERMISSIONS (on the filesystem itself):                   │
│    ├── Standard Linux permissions (owner/group/other)           │
│    ├── UID/GID-based access                                      │
│    └── ⚡ UIDs must match across VMs! If VM-1 user has UID 1000 │
│        and VM-2 same user has UID 1001, they're different users│
│                                                                       │
│ Shared VPC access:                                                  │
│ ├── Filestore instance in Shared VPC host project               │
│ ├── VMs in service projects can mount via shared subnets       │
│ └── IP-based access rules still apply                           │
│                                                                       │
│ Cross-VPC access (VPC Peering):                                    │
│ ├── Filestore in VPC-A, VM in VPC-B                             │
│ ├── VPC Peering between VPC-A and VPC-B                         │
│ ├── VM in VPC-B mounts Filestore using its private IP          │
│ └── Ensure peering exports/imports custom routes               │
│                                                                       │
│ On-premises access:                                                 │
│ ├── Via Cloud VPN or Cloud Interconnect                         │
│ ├── On-prem client mounts Filestore's private IP               │
│ └── Latency depends on VPN/Interconnect link                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Filestore Multishares for GKE

```
┌─────────────────────────────────────────────────────────────────────┐
│           FILESTORE MULTISHARES FOR GKE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Each GKE PVC with Filestore CSI driver creates a NEW     │
│ Filestore instance (min 1 TB). If you have 20 PVCs needing       │
│ 50 GB each, you'd get 20 × 1 TB = 20 TB billed — huge waste!   │
│                                                                       │
│ Solution: Multishares — one Filestore Enterprise instance with   │
│ MULTIPLE file shares, each backing a different PVC.              │
│                                                                       │
│ ┌─────────────────────────────────────────────────────┐            │
│ │  Filestore Enterprise Instance (10 TB)               │            │
│ │                                                       │            │
│ │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │            │
│ │  │ Share 1  │ │ Share 2  │ │ Share 3  │ │Share N │ │            │
│ │  │ 100 GB   │ │ 200 GB   │ │ 50 GB    │ │ ...    │ │            │
│ │  │ (PVC-A)  │ │ (PVC-B)  │ │ (PVC-C)  │ │        │ │            │
│ │  └──────────┘ └──────────┘ └──────────┘ └────────┘ │            │
│ │  ⚡ Up to 10 shares per Enterprise instance         │            │
│ └─────────────────────────────────────────────────────┘            │
│     ▲             ▲             ▲                                  │
│     │             │             │                                  │
│ ┌───┴──┐     ┌───┴──┐     ┌───┴──┐                                │
│ │Pod A │     │Pod B │     │Pod C │                                │
│ └──────┘     └──────┘     └──────┘                                │
│                                                                       │
│ Benefits:                                                           │
│ ├── Cost: One instance shared across many PVCs (not 1:1)        │
│ ├── Min PVC: 100 GiB (vs 1 TB per instance without Multishares)│
│ ├── Dynamic: CSI driver manages share creation automatically    │
│ └── Enterprise tier: Regional HA, NFSv4.1                       │
│                                                                       │
│ Setup:                                                              │
│ # StorageClass for Multishares                                    │
│ apiVersion: storage.k8s.io/v1                                     │
│ kind: StorageClass                                                  │
│ metadata:                                                           │
│   name: filestore-multishare                                       │
│ provisioner: filestore.csi.storage.gke.io                         │
│ parameters:                                                         │
│   tier: enterprise                                                 │
│   multishare: "true"                                               │
│   max-volume-size: "128Gi"                                        │
│   network: default                                                 │
│ volumeBindingMode: Immediate                                      │
│ allowVolumeExpansion: true                                         │
│                                                                       │
│ ⚡ The CSI driver auto-creates Filestore Enterprise instances     │
│   and allocates shares. It packs PVCs into existing instances   │
│   before creating new ones.                                      │
│                                                                       │
│ AWS equivalent: EFS Access Points (multiple mount targets from   │
│   one EFS instance — similar concept)                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Monitoring & Troubleshooting

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING & TROUBLESHOOTING                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Monitoring metrics for Filestore:                             │
│ ├── file.googleapis.com/nfs/server/used_bytes_percent            │
│ │   ⚡ CRITICAL — alert at 80% to avoid running out of space     │
│ │                                                                 │
│ ├── file.googleapis.com/nfs/server/read_ops_count                │
│ ├── file.googleapis.com/nfs/server/write_ops_count               │
│ ├── file.googleapis.com/nfs/server/read_bytes_count              │
│ ├── file.googleapis.com/nfs/server/write_bytes_count             │
│ └── file.googleapis.com/nfs/server/free_bytes                    │
│                                                                       │
│ Recommended alerts:                                                 │
│ ├── Disk usage > 80%: Warning — resize soon!                    │
│ ├── Disk usage > 90%: Critical — resize immediately!            │
│ ├── IOPS sustained at max: Upgrade tier or scale capacity       │
│ └── Backup failure: Check backup job logs                       │
│                                                                       │
│ Common issues:                                                      │
│ ┌──────────────────────┬────────────────────────────────────┐     │
│ │ Problem              │ Solution                            │     │
│ ├──────────────────────┼────────────────────────────────────┤     │
│ │ Cannot mount         │ Check firewall rules (port 2049)   │     │
│ │                      │ Check VPC (same network?)           │     │
│ │                      │ Check IP access rules on instance  │     │
│ │                      │ Install nfs-common / nfs-utils     │     │
│ │                      │                                     │     │
│ │ Permission denied    │ Check POSIX permissions (UID/GID)  │     │
│ │                      │ Check root squash setting           │     │
│ │                      │ Check access mode (read-only?)     │     │
│ │                      │                                     │     │
│ │ Slow performance     │ Check VM machine type (network     │     │
│ │                      │ bandwidth limit)                    │     │
│ │                      │ Check mount options (rsize/wsize)  │     │
│ │                      │ Check tier — Basic SSD has fixed   │     │
│ │                      │ IOPS regardless of capacity        │     │
│ │                      │ Use Zonal tier for high perf       │     │
│ │                      │                                     │     │
│ │ Out of space         │ Scale up capacity (online resize)  │     │
│ │                      │ Delete old snapshots (they use     │     │
│ │                      │ instance capacity!)                 │     │
│ │                      │ Clean up unneeded files            │     │
│ │                      │                                     │     │
│ │ Stale NFS handle     │ Remount: umount -f && mount        │     │
│ │                      │ Filestore may have been recreated  │     │
│ └──────────────────────┴────────────────────────────────────┘     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Terraform & CLI

### Terraform

```hcl
# === Basic SSD Filestore Instance ===
resource "google_filestore_instance" "shared_data" {
  name     = "shared-data-fs"
  location = "asia-south1-a"   # Zone for Basic/Zonal tiers
  tier     = "BASIC_SSD"

  file_shares {
    name       = "vol1"
    capacity_gb = 2560   # 2.5 TB (min for Basic SSD)

    # NFS export options (access control)
    nfs_export_options {
      ip_ranges   = ["10.0.0.0/8"]
      access_mode = "READ_WRITE"
      squash_mode = "ROOT_SQUASH"   # or NO_ROOT_SQUASH
    }

    nfs_export_options {
      ip_ranges   = ["10.100.0.0/24"]
      access_mode = "READ_ONLY"
      squash_mode = "ROOT_SQUASH"
    }
  }

  networks {
    network      = "default"
    modes        = ["MODE_IPV4"]
    connect_mode = "DIRECT_PEERING"   # or PRIVATE_SERVICE_ACCESS
  }

  labels = {
    environment = "prod"
    team        = "web"
  }
}

# === Zonal (high-performance) ===
resource "google_filestore_instance" "high_perf" {
  name     = "high-perf-fs"
  location = "asia-south1-a"
  tier     = "ZONAL"

  file_shares {
    name        = "data"
    capacity_gb = 10240   # 10 TB

    nfs_export_options {
      ip_ranges   = ["10.0.0.0/8"]
      access_mode = "READ_WRITE"
      squash_mode = "NO_ROOT_SQUASH"
    }
  }

  networks {
    network = "default"
    modes   = ["MODE_IPV4"]
  }
}

# === Enterprise (Regional HA) ===
resource "google_filestore_instance" "enterprise" {
  name     = "enterprise-fs"
  location = "asia-south1"   # REGION for Enterprise/Regional tiers
  tier     = "ENTERPRISE"

  file_shares {
    name        = "share1"
    capacity_gb = 1024   # 1 TB

    nfs_export_options {
      ip_ranges   = ["10.0.0.0/8"]
      access_mode = "READ_WRITE"
      squash_mode = "ROOT_SQUASH"
    }
  }

  networks {
    network = "default"
    modes   = ["MODE_IPV4"]
  }

  # CMEK encryption
  kms_key_name = google_kms_crypto_key.filestore_key.id
}

# === Backup ===
resource "google_filestore_backup" "daily" {
  name              = "daily-backup-20260517"
  location          = "asia-south1"
  source_instance   = google_filestore_instance.shared_data.id
  source_file_share = "vol1"

  labels = {
    purpose = "daily-backup"
  }
}

# === Snapshot (Zonal/Regional/Enterprise only) ===
resource "google_filestore_snapshot" "pre_deploy" {
  name     = "pre-deploy-snap"
  instance = google_filestore_instance.high_perf.name
  location = "asia-south1-a"

  labels = {
    purpose = "pre-deployment"
  }
}

# Output the Filestore IP for mounting
output "filestore_ip" {
  value = google_filestore_instance.shared_data.networks[0].ip_addresses[0]
}
```

### gcloud CLI Reference

```bash
# =============================================
# CREATE FILESTORE INSTANCES
# =============================================

# Basic SSD instance
gcloud filestore instances create shared-data-fs \
  --tier=BASIC_SSD \
  --file-share=name=vol1,capacity=2560GB \
  --network=name=default \
  --zone=asia-south1-a

# Zonal instance (high performance)
gcloud filestore instances create high-perf-fs \
  --tier=ZONAL \
  --file-share=name=data,capacity=10TB \
  --network=name=default \
  --zone=asia-south1-a

# Enterprise instance (regional HA)
gcloud filestore instances create enterprise-fs \
  --tier=ENTERPRISE \
  --file-share=name=share1,capacity=1TB \
  --network=name=default \
  --region=asia-south1

# With CMEK encryption
gcloud filestore instances create encrypted-fs \
  --tier=ENTERPRISE \
  --file-share=name=vol1,capacity=1TB \
  --network=name=default \
  --region=asia-south1 \
  --kms-key=projects/proj/locations/asia-south1/keyRings/kr/cryptoKeys/key

# =============================================
# MANAGE INSTANCES
# =============================================

# List instances
gcloud filestore instances list

# Describe instance (get IP address!)
gcloud filestore instances describe shared-data-fs \
  --zone=asia-south1-a
# Look for: ipAddresses: ['10.x.x.x']

# Resize (scale up or down for Zonal/Regional/Enterprise)
gcloud filestore instances update shared-data-fs \
  --file-share=name=vol1,capacity=5TB \
  --zone=asia-south1-a

# Update access control
gcloud filestore instances update shared-data-fs \
  --zone=asia-south1-a \
  --file-share=name=vol1,capacity=2560GB,\
nfs-export-options=ip-ranges="10.0.1.0/24",access-mode=READ_WRITE,squash-mode=ROOT_SQUASH

# Delete instance
gcloud filestore instances delete shared-data-fs \
  --zone=asia-south1-a

# =============================================
# BACKUPS
# =============================================

# Create backup
gcloud filestore backups create daily-backup-20260517 \
  --instance=shared-data-fs \
  --file-share=vol1 \
  --instance-zone=asia-south1-a \
  --region=asia-south1

# List backups
gcloud filestore backups list --region=asia-south1

# Describe backup
gcloud filestore backups describe daily-backup-20260517 \
  --region=asia-south1

# Create new instance from backup
gcloud filestore instances create restored-fs \
  --tier=BASIC_SSD \
  --file-share=name=vol1,capacity=2560GB,\
source-backup=daily-backup-20260517,source-backup-region=asia-south1 \
  --network=name=default \
  --zone=asia-south1-b

# Delete backup
gcloud filestore backups delete daily-backup-20260517 \
  --region=asia-south1

# =============================================
# SNAPSHOTS (Zonal/Regional/Enterprise only)
# =============================================

# Create snapshot
gcloud filestore instances snapshots create snap-20260517 \
  --instance=high-perf-fs \
  --zone=asia-south1-a

# List snapshots
gcloud filestore instances snapshots list \
  --instance=high-perf-fs \
  --zone=asia-south1-a

# Revert to snapshot (⚠️ replaces current data!)
gcloud filestore instances revert high-perf-fs \
  --target-snapshot=snap-20260517 \
  --zone=asia-south1-a

# Delete snapshot
gcloud filestore instances snapshots delete snap-20260517 \
  --instance=high-perf-fs \
  --zone=asia-south1-a
```

---

## Part 12: Real-World Patterns

### Startup

```
┌─────────────────────────────────────────────────────────────────────┐
│ STARTUP (5-10 developers)                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Filestore:                                                           │
│ ├── Basic HDD — 1 TB for shared dev/staging assets              │
│ ├── One instance shared by all app VMs                           │
│ ├── Mounted at /mnt/shared for user uploads, media files        │
│ └── Cost: ~$200/month (1 TB Basic HDD)                          │
│                                                                       │
│ Backups:                                                             │
│ ├── Weekly manual backup (or script via cron + gcloud)          │
│ └── Keep last 4 backups (1 month)                                │
│                                                                       │
│ No GKE integration needed (simple VM setup)                       │
│ Google-managed encryption (free)                                  │
│                                                                       │
│ ⚡ Alternative: For very small scale, consider Cloud Storage       │
│   + FUSE mount (gcsfuse) — cheaper but slower, no POSIX locks.  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Mid-Size

```
┌─────────────────────────────────────────────────────────────────────┐
│ MID-SIZE (50-100 developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Filestore:                                                           │
│ ├── Zonal (Basic SSD for budget-conscious teams)                │
│ ├── 5-10 TB for CMS media, shared app data                     │
│ ├── IP-based access control per subnet:                         │
│ │   ├── App subnet: read-write                                  │
│ │   ├── Reporting subnet: read-only                             │
│ │   └── Admin VM: admin (no root squash)                       │
│ └── Root squash enabled for app servers                         │
│                                                                       │
│ GKE:                                                                │
│ ├── Filestore CSI driver for shared PVCs (ReadWriteMany)       │
│ ├── WordPress/Drupal pods sharing media directory               │
│ └── Consider Multishares if many small PVCs needed             │
│                                                                       │
│ Backups:                                                             │
│ ├── Daily automated backups (script or Cloud Scheduler)        │
│ ├── 30-day retention                                             │
│ └── Snapshots before deployments (Zonal tier)                  │
│                                                                       │
│ Monitoring:                                                         │
│ ├── Alert at 80% capacity usage                                 │
│ └── Scale up proactively                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Enterprise

```
┌─────────────────────────────────────────────────────────────────────┐
│ ENTERPRISE (500+ developers)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Filestore:                                                           │
│ ├── Enterprise tier for mission-critical file shares            │
│ │   (Regional HA, NFSv4.1, snapshots)                           │
│ ├── Zonal tier for high-performance workloads (media, HPC)     │
│ ├── Separate instances per environment (prod/staging/dev)       │
│ ├── Shared VPC: Filestore in host project, access from services│
│ └── On-prem access via Cloud Interconnect                       │
│                                                                       │
│ GKE:                                                                │
│ ├── Filestore Multishares (Enterprise) for PVC efficiency      │
│ │   → One instance serves 10+ PVCs                             │
│ ├── Separate StorageClasses per team/workload type             │
│ └── ReadWriteMany PVCs for shared data across pods             │
│                                                                       │
│ Security:                                                           │
│ ├── CMEK encryption on all instances                            │
│ ├── VPC Service Controls perimeter around Filestore            │
│ ├── Root squash enabled everywhere                              │
│ ├── Strict IP-based access rules per subnet/team               │
│ └── Data access audit logs                                      │
│                                                                       │
│ Backups:                                                             │
│ ├── Automated daily backups (Terraform + Cloud Scheduler)      │
│ ├── 90-day retention for compliance                             │
│ ├── Quarterly DR restore tests                                  │
│ └── Snapshots every 4 hours for RPO < 4h                       │
│                                                                       │
│ Monitoring:                                                         │
│ ├── Capacity alerts at 70%, 80%, 90%                           │
│ ├── IOPS/throughput dashboards per instance                    │
│ ├── Backup job success/failure alerts                           │
│ └── Integration with PagerDuty / Opsgenie                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌──────────────────────────────────────────────────────────────────────┐
│ FILESTORE — QUICK REFERENCE                                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│ What: Managed NFS file server (shared filesystem)                    │
│ Protocol: NFSv3 (all tiers), NFSv4.1 (Enterprise only)             │
│ Access: Multiple VMs read + write simultaneously                    │
│                                                                        │
│ Tiers:                                                                │
│ ├── Basic HDD: 1-63 TB, cheapest, dev/test                          │
│ ├── Basic SSD: 2.5-63 TB, fixed IOPS, general purpose               │
│ ├── Zonal: 1-100 TB, high perf, scale up/down ✅                     │
│ ├── Regional: 1-100 TB, multi-zone HA, scale up/down                │
│ └── Enterprise: 1-10 TB, HA + NFSv4.1 + Multishares                 │
│                                                                        │
│ Capacity: Pre-provisioned (pay for allocated, not used!)            │
│ Resize: Basic = up only | Zonal/Regional/Enterprise = up & down     │
│                                                                        │
│ Backups: Full copy, independent, survives instance deletion          │
│ Snapshots: Incremental, on-instance, Zonal/Regional/Enterprise only │
│ Encryption: Google-managed (default) or CMEK                        │
│                                                                        │
│ GKE: CSI driver (dynamic provisioning) or manual PV/PVC             │
│ Multishares: Multiple PVCs from one Enterprise instance              │
│                                                                        │
│ Mount: sudo mount IP:/share /mnt/point                               │
│ Access control: IP-based rules + POSIX permissions + root squash    │
│                                                                        │
│ AWS ↔ GCP mapping:                                                     │
│ ├── EFS Standard → Basic SSD / Zonal                                  │
│ ├── EFS Infrequent → Basic HDD                                        │
│ ├── EFS (regional) → Regional                                         │
│ ├── EFS Access Points → Multishares (Enterprise)                      │
│ └── EFS backup → Filestore Backup                                     │
│                                                                        │
│ Azure ↔ GCP mapping:                                                   │
│ ├── Azure Files (NFS) → Basic SSD / Zonal                             │
│ ├── Azure Files Premium → Zonal / Regional                            │
│ ├── Azure NetApp Files → Enterprise                                    │
│ └── Azure Files ZRS → Regional                                         │
│                                                                        │
│ Key difference from AWS EFS:                                          │
│ ├── EFS: Elastic (pay per used GB) — auto-scales                     │
│ ├── Filestore: Pre-provisioned (pay for allocated capacity)          │
│ └── Filestore needs manual capacity planning!                        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## What is NFS? (Beginner Explanation)

```
┌─────────────────────────────────────────────────────────────────────┐
│           NFS — NETWORK FILE SYSTEM                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ NFS = A shared folder on a network drive that multiple computers  │
│       can read and write to at the same time.                      │
│                                                                       │
│ 📁 Office Shared Drive Analogy:                                     │
│ ├── Imagine a shared folder on your office network (like a       │
│ │   mapped drive \\server\shared on Windows)                    │
│ ├── Everyone in the office can open, edit, and save files there │
│ ├── If you save a file, your colleague sees it immediately      │
│ ├── NFS is the LINUX version of this — same concept, different  │
│ │   protocol                                                     │
│ └── Filestore = Google manages the NFS server for you           │
│                                                                       │
│ Without NFS (Persistent Disk only):                                │
│ ┌──────┐    ┌──────┐    ┌──────┐                                  │
│ │ VM 1 │    │ VM 2 │    │ VM 3 │                                  │
│ │┌────┐│    │┌────┐│    │┌────┐│                                  │
│ ││Disk││    ││Disk││    ││Disk││ ← Each VM has its OWN disk       │
│ │└────┘│    │└────┘│    │└────┘│   Data is NOT shared!             │
│ └──────┘    └──────┘    └──────┘                                  │
│                                                                       │
│ With NFS (Filestore):                                               │
│ ┌──────┐    ┌──────┐    ┌──────┐                                  │
│ │ VM 1 │    │ VM 2 │    │ VM 3 │                                  │
│ └──┬───┘    └──┬───┘    └──┬───┘                                  │
│    │           │           │                                      │
│    └───────────┼───────────┘                                      │
│         ┌──────▼──────┐                                           │
│         │  FILESTORE   │ ← ONE shared folder                     │
│         │  /vol1       │   ALL VMs read/write the same data!     │
│         └─────────────┘                                           │
│                                                                       │
│ ── WHY FILESTORE MATTERS ──                                         │
│                                                                       │
│ 1. Shared storage for multiple VMs:                                │
│    ├── Web servers sharing uploaded images/files                 │
│    ├── CMS platforms (WordPress) where all instances need the    │
│    │   same content (themes, plugins, uploads)                   │
│    └── Configuration files shared across a fleet of servers      │
│                                                                       │
│ 2. Shared storage for GKE (Kubernetes):                            │
│    ├── Multiple pods need access to the same files               │
│    ├── Persistent Disk = ReadWriteOnce (one pod at a time)      │
│    ├── Filestore = ReadWriteMany (all pods read + write) ✅      │
│    └── Common for media processing, ML pipelines, shared logs   │
│                                                                       │
│ 3. When you need POSIX file semantics:                             │
│    ├── Applications that expect a real filesystem (not an API)  │
│    ├── File locking, symlinks, permissions, directory structure  │
│    └── Cloud Storage (GCS) is NOT a filesystem — no locking,    │
│        no POSIX, high latency. Filestore fills this gap.         │
│                                                                       │
│ ⚡ If only ONE VM needs storage → use Persistent Disk (cheaper).  │
│ ⚡ If MULTIPLE VMs need shared storage → use Filestore.            │
│ ⚡ If you need unlimited cheap storage via API → use Cloud Storage.│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Instances

### Resizing Filestore Capacity

```
Console → Filestore → Instances → [click your instance]
→ Edit

┌─────────────────────────────────────────────────────────────────────┐
│           RESIZE FILESTORE CAPACITY                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Current capacity: [2.5] TB                                         │
│ New capacity: [5] TB                                               │
│                                                                       │
│ [Save]                                                               │
│                                                                       │
│ Resize rules by tier:                                               │
│                                                                       │
│ Basic HDD / Basic SSD:                                              │
│ ├── ✅ Can INCREASE capacity (scale up)                            │
│ ├── ❌ Cannot DECREASE capacity (no scale down)                    │
│ ├── Online — no downtime, no unmount needed                      │
│ └── Takes a few minutes to complete                               │
│                                                                       │
│ Zonal / Regional / Enterprise:                                     │
│ ├── ✅ Can INCREASE capacity (scale up)                            │
│ ├── ✅ Can DECREASE capacity (scale down!) ← big advantage       │
│ ├── Cannot go below minimum tier size (1 TB)                     │
│ ├── Cannot go below currently USED capacity                      │
│ ├── Online — no downtime, no unmount needed                      │
│ └── ⚡ Scale down to save costs when demand drops!                │
│                                                                       │
│ ⚡ Unlike Persistent Disk, Zonal/Regional Filestore CAN shrink.   │
│ ⚡ No filesystem resize needed — NFS handles it automatically.    │
│   Clients see the new capacity immediately.                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Deleting a Filestore Instance

```
Console → Filestore → Instances → [select instance checkbox]
→ Delete (top bar)

┌─────────────────────────────────────────────────────────────────────┐
│           DELETE FILESTORE INSTANCE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️  "Are you sure you want to delete 'shared-data-fs'?"             │
│                                                                       │
│ What happens:                                                       │
│ ├── ALL data in the file share is PERMANENTLY deleted            │
│ ├── All file shares on this instance are deleted                 │
│ ├── This action CANNOT be undone                                  │
│ ├── Existing backups are NOT deleted                              │
│ │   (backups are independent resources — you keep them)          │
│ └── VMs that have this mounted will get NFS errors               │
│     (mount point becomes stale)                                  │
│                                                                       │
│ Before deleting:                                                    │
│ ├── Create a backup if you might need the data later             │
│ ├── Unmount on all VMs first (sudo umount /mnt/shared)          │
│ ├── Remove /etc/fstab entries on all VMs                         │
│ └── Update GKE PV/PVC if used with Kubernetes                   │
│                                                                       │
│ ⚡ Take a backup before deleting — Filestore backups can be used  │
│   to create a new instance with the same data later.              │
│                                                                       │
│ [Delete]   [Cancel]                                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating & Restoring Backups

```
┌─────────────────────────────────────────────────────────────────────┐
│           FILESTORE BACKUPS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── CREATE A BACKUP ──                                               │
│                                                                       │
│ Console → Filestore → Backups → Create Backup                     │
│                                                                       │
│ Source instance: [shared-data-fs ▼]                                │
│ Source file share: [vol1 ▼]                                        │
│ Region: [asia-south1] (auto-filled from source)                   │
│ Backup ID: [shared-data-backup-20260522]                           │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ ⚡ Backups are full copies (not incremental like PD snapshots).    │
│ ⚡ Backup is stored in the same region as the source instance.    │
│ ⚡ You can create backups while instance is running — no downtime. │
│ ⚡ Backups are independent — survive even if you delete the       │
│   source instance.                                                 │
│                                                                       │
│ ── RESTORE A BACKUP ──                                              │
│                                                                       │
│ Option 1: Restore to a NEW Filestore instance                      │
│ Console → Filestore → Backups → [select backup]                   │
│ → Restore → Create new instance                                   │
│                                                                       │
│ Instance ID: [shared-data-fs-restored]                             │
│ Tier: [same or different tier]                                     │
│ Capacity: [>= backup size]                                        │
│ Network: [your VPC]                                                │
│ [Restore]                                                           │
│                                                                       │
│ ⚡ Creates a brand new Filestore instance with the backup data.    │
│ ⚡ Takes several minutes depending on backup size.                 │
│                                                                       │
│ Option 2: Restore to the EXISTING instance (overwrite!)           │
│ Console → Filestore → Instances → [your instance]                 │
│ → Restore backup → [select backup]                                │
│                                                                       │
│ ⚠️  This OVERWRITES all current data in the file share!             │
│ Any files added after the backup was taken will be LOST.          │
│                                                                       │
│ ⚡ Restore to existing instance is faster than creating a new one. │
│ ⚡ All mounted clients see the restored data immediately.          │
│                                                                       │
│ ── DELETE A BACKUP ──                                               │
│                                                                       │
│ Console → Filestore → Backups → [select backup] → Delete         │
│ ⚡ Deleting a backup does NOT affect the source instance.          │
│ ⚡ Backups you no longer need should be deleted to save costs.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 24: Cloud SQL** → `24-cloud-sql.md`
