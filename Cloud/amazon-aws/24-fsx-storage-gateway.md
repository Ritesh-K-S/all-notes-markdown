# Chapter 24: FSx & Storage Gateway

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Amazon FSx Overview](#part-1-amazon-fsx-overview)
- [Part 2: FSx for Windows File Server (Full Portal Walkthrough)](#part-2-fsx-for-windows-file-server-full-portal-walkthrough)
- [Part 3: FSx for Lustre (Full Portal Walkthrough)](#part-3-fsx-for-lustre-full-portal-walkthrough)
- [Part 4: FSx for NetApp ONTAP](#part-4-fsx-for-netapp-ontap)
- [Part 5: FSx for OpenZFS](#part-5-fsx-for-openzfs)
- [Part 6: AWS Storage Gateway](#part-6-aws-storage-gateway)
- [Part 7: Storage Gateway Types (Full Portal Walkthrough)](#part-7-storage-gateway-types-full-portal-walkthrough)
- [Part 8: Terraform & CLI Examples](#part-8-terraform--cli-examples)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is FSx? Why Not Just Use EFS?

EFS is great for Linux servers that need shared storage, but what if:
- Your company uses **Windows servers** that need shared drives with Active Directory? → **FSx for Windows**
- You're training **ML models** or running **supercomputer simulations** that need blazing-fast file I/O? → **FSx for Lustre**
- You need enterprise features like **multi-protocol access** (NFS + SMB + iSCSI)? → **FSx for NetApp ONTAP**

**FSx** = Fully managed file systems built on popular technologies. Think of it as: *"I need a specific type of file server, and I don't want to manage it myself."*

### What is Storage Gateway? Why Do We Need It?

Imagine your company has 50 TB of data on-premises and can't move everything to S3 overnight. **Storage Gateway** is a bridge between your on-premises data center and AWS cloud storage. It lets your existing servers keep using their local file shares (NFS/SMB) while **automatically syncing data to S3 in the background**.

**Think of it as a two-way mirror** — your on-premises apps see local storage, but the data actually lives in (or is backed up to) AWS.

Amazon FSx provides fully managed file systems built on popular technologies (Windows, Lustre, NetApp, ZFS). AWS Storage Gateway bridges on-premises environments with AWS cloud storage. Together, they cover enterprise and hybrid storage needs.

```
What you'll learn:
├── FSx for Windows File Server
│   ├── SMB protocol, Active Directory integration
│   ├── Single-AZ vs Multi-AZ deployment
│   └── Full portal walkthrough
├── FSx for Lustre
│   ├── High-performance computing (HPC)
│   ├── S3 data repository integration
│   └── Full portal walkthrough
├── FSx for NetApp ONTAP
│   ├── Multi-protocol (NFS, SMB, iSCSI)
│   └── Data management features
├── FSx for OpenZFS
│   ├── NFS protocol, snapshots, compression
│   └── Linux workload migration
├── AWS Storage Gateway
│   ├── S3 File Gateway
│   ├── FSx File Gateway
│   ├── Volume Gateway (Stored / Cached)
│   └── Tape Gateway
└── Decision guide: When to use which
```

---

## Part 1: Amazon FSx Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           FSx FAMILY COMPARISON                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┬──────────────┬──────────────┬──────────────────┐  │
│ │ FSx Type     │ Protocol     │ OS Support   │ Best For          │  │
│ ├──────────────┼──────────────┼──────────────┼──────────────────┤  │
│ │ Windows File │ SMB          │ Windows      │ Windows apps,     │  │
│ │ Server       │              │ (+ Linux)    │ AD integration,   │  │
│ │              │              │              │ SQL Server        │  │
│ │              │              │              │                   │  │
│ │ Lustre       │ POSIX/Lustre │ Linux        │ HPC, ML training, │  │
│ │              │              │              │ media processing, │  │
│ │              │              │              │ financial modeling│  │
│ │              │              │              │                   │  │
│ │ NetApp ONTAP │ NFS, SMB,    │ Linux,       │ Multi-protocol,   │  │
│ │              │ iSCSI        │ Windows      │ enterprise        │  │
│ │              │              │              │ migration         │  │
│ │              │              │              │                   │  │
│ │ OpenZFS      │ NFS          │ Linux        │ ZFS migration,    │  │
│ │              │              │              │ snapshots,        │  │
│ │              │              │              │ compression       │  │
│ └──────────────┴──────────────┴──────────────┴──────────────────┘  │
│                                                                       │
│ Decision guide:                                                      │
│ ├── Windows + SMB + Active Directory → FSx for Windows           │
│ ├── HPC / ML / massive throughput → FSx for Lustre              │
│ ├── Multi-protocol (NFS + SMB + iSCSI) → FSx for NetApp ONTAP  │
│ ├── Linux + ZFS features → FSx for OpenZFS                      │
│ ├── Shared Linux NFS (general) → Use EFS instead                │
│ └── Object storage → Use S3                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: FSx for Windows File Server (Full Portal Walkthrough)

```
Console → FSx → Create file system → Amazon FSx for Windows File Server

┌─────────────────────────────────────────────────────────────────┐
│           CREATE FSx for WINDOWS FILE SERVER                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── File system details ──                                      │
│                                                                   │
│ File system name: [corp-shared-drive]                          │
│                                                                   │
│ Deployment type:                                                │
│   ● Multi-AZ (recommended for production)                      │
│   ○ Single-AZ 2                                                │
│   ○ Single-AZ 1                                                │
│                                                                   │
│ → Multi-AZ: Active + standby file servers in different AZs.  │
│   Automatic failover. Higher availability. 2x cost.           │
│ → Single-AZ 2: One AZ, latest gen. Dev/test.                 │
│ → Single-AZ 1: One AZ, older gen. Legacy.                    │
│                                                                   │
│ Storage type:                                                   │
│   ● SSD    ○ HDD                                               │
│ → SSD: Low latency, higher IOPS. Databases, active shares.   │
│ → HDD: Higher throughput, lower cost. Home dirs, archives.    │
│                                                                   │
│ Storage capacity (GiB): [1024]                                │
│ → SSD: 32 GiB - 65,536 GiB                                   │
│ → HDD: 2,000 GiB - 65,536 GiB                                │
│                                                                   │
│ Throughput capacity (MB/s): [256]                              │
│ → Drives network + disk I/O performance                       │
│ → Options: 8, 16, 32, 64, 128, 256, 512, 1024, 2048 MB/s    │
│ → Higher = more CPU, memory, and network for file server     │
│                                                                   │
│ ── Network & security ──                                       │
│                                                                   │
│ VPC: [vpc-main ▼]                                              │
│ Preferred subnet: [subnet-priv-1a ▼]                          │
│ Standby subnet: [subnet-priv-1b ▼]  (Multi-AZ only)          │
│ Security groups: [sg-fsx-access ▼]                            │
│ → Allow SMB (TCP 445) + related ports from clients           │
│                                                                   │
│ ── Windows authentication ──                                   │
│                                                                   │
│ ● AWS Managed Microsoft AD                                     │
│ ○ Self-managed Microsoft AD                                    │
│                                                                   │
│ Directory: [corp.example.com ▼]                                │
│ → Integrates with Active Directory for authentication         │
│ → NTFS permissions, user/group access control                │
│ → Required: Must have an AD (managed or self-managed)        │
│                                                                   │
│ For self-managed AD:                                           │
│   Domain name: [corp.example.com]                             │
│   DNS IPs: [10.0.1.10, 10.0.2.10]                           │
│   Service account username: [FSxServiceAccount]               │
│   Service account password: [********]                        │
│   OU: [OU=FSx,DC=corp,DC=example,DC=com]                    │
│                                                                   │
│ ── Encryption ──                                                │
│ KMS key: [aws/fsx (default) ▼]                                │
│                                                                   │
│ ── Maintenance & backup ──                                     │
│ Automatic daily backups: ☑ Enabled                            │
│ Backup window: [01:00 - 02:00 UTC ▼]                         │
│ Backup retention: [7] days                                    │
│ Maintenance window: [Sun 04:00 - Sun 04:30 ▼]                │
│                                                                   │
│ ── Tags ──                                                      │
│ Key: [environment]  Value: [production]                       │
│                                                                   │
│                    [Create file system]                        │
└─────────────────────────────────────────────────────────────────┘

Key features:
├── DFS (Distributed File System) Namespaces — scale beyond 64 TB
├── Shadow Copies — user-initiated restore of previous versions
├── Data deduplication — reduce storage consumption
├── User quotas — limit storage per user
└── ⚡ DNS: \\fs-abc123.corp.example.com\share
```

---

## Part 3: FSx for Lustre (Full Portal Walkthrough)

```
Console → FSx → Create file system → Amazon FSx for Lustre

┌─────────────────────────────────────────────────────────────────┐
│           CREATE FSx for LUSTRE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ File system name: [ml-training-data]                           │
│                                                                   │
│ Deployment type:                                                │
│   ○ Persistent 2: Long-term storage, HA, data replication.    │
│   ● Scratch 2: Temporary, highest throughput, no replication. │
│                                                                   │
│ → Persistent: Data replicated within AZ. For long-term data. │
│   Use for: workloads running days/weeks.                      │
│ → Scratch: No replication. Highest burst throughput.          │
│   Use for: short-term processing, can be recreated from S3.  │
│                                                                   │
│ Storage capacity (GiB): [4800]                                │
│ → Increments of 1,200 GiB (Scratch) or 1,200/2,400 GiB     │
│                                                                   │
│ Throughput per unit of storage:                                │
│   [200 ▼] MB/s/TiB  (Persistent only)                        │
│   Options: 125, 250, 500, 1000 MB/s/TiB                      │
│                                                                   │
│ ── Data repository (S3 integration) ──                         │
│ Import path: [s3://ml-datasets/training/]                     │
│ → Lustre lazily loads data from S3 on first access            │
│ → File metadata imported immediately                          │
│                                                                   │
│ Export path: [s3://ml-datasets/results/]                       │
│ → Write results back to S3 using hsm_archive command          │
│                                                                   │
│ Import policy: ☑ New ☑ Changed ☑ Deleted                     │
│ → Auto-import changes from S3 to Lustre                       │
│                                                                   │
│ ── Network ──                                                   │
│ VPC: [vpc-main ▼]                                              │
│ Subnet: [subnet-priv-1a ▼]  (single subnet only)             │
│ Security groups: [sg-lustre ▼]                                │
│ → Allow TCP 988 + 1018-1023 (Lustre protocol)                │
│                                                                   │
│ ── Encryption ──                                                │
│ KMS key: [aws/fsx ▼]                                          │
│                                                                   │
│                    [Create file system]                        │
└─────────────────────────────────────────────────────────────────┘

Performance:
├── Scratch 2: 200 MB/s/TiB baseline, 1,300 MB/s/TiB burst
├── Persistent: 125-1000 MB/s/TiB (configurable)
├── Sub-millisecond latency
├── Example: 4.8 TB Scratch = ~960 MB/s baseline, ~6.24 GB/s burst
└── ⚡ Perfect for ML training data (link S3 dataset → Lustre → GPU)
```

---

## Part 4: FSx for NetApp ONTAP

```
┌─────────────────────────────────────────────────────────────────────┐
│           FSx for NETAPP ONTAP                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Fully managed NetApp ONTAP file system. Multi-protocol     │
│ (NFS, SMB, iSCSI) with enterprise data management features.       │
│                                                                       │
│ Key features:                                                        │
│ ├── Multi-protocol: NFS + SMB + iSCSI simultaneously            │
│ ├── Data tiering: Auto-tier cold data to cheaper capacity pool  │
│ ├── Snapshots: Instant, space-efficient                           │
│ ├── FlexClone: Instant clones for dev/test                       │
│ ├── SnapMirror: Cross-region replication                         │
│ ├── Deduplication + compression: Reduce storage 50-65%          │
│ └── Multi-AZ deployment for HA                                    │
│                                                                       │
│ Architecture:                                                        │
│ ├── File System → contains SVMs                                  │
│ ├── SVM (Storage Virtual Machine) → logical file server         │
│ ├── Volume → data container within SVM                           │
│ └── LUN → block storage within volume (for iSCSI)               │
│                                                                       │
│ Best for:                                                            │
│ ├── Migrating on-premises NetApp to AWS                          │
│ ├── Multi-protocol workloads (Linux + Windows)                   │
│ ├── Enterprise applications (Oracle, SAP, SQL Server)           │
│ └── Dev/test with instant cloning                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: FSx for OpenZFS

```
┌─────────────────────────────────────────────────────────────────────┐
│           FSx for OPENZFS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed ZFS file system. NFS protocol with ZFS features.    │
│                                                                       │
│ Key features:                                                        │
│ ├── Up to 1 million IOPS, sub-ms latency                         │
│ ├── Snapshots: Instant, user-accessible via .zfs/snapshot       │
│ ├── Data compression: LZ4, ZSTD (up to 3-4x reduction)         │
│ ├── Clones: Instant writable copies from snapshots               │
│ ├── Copy-on-write: Efficient data management                     │
│ └── NFS protocol (v3, v4.0, v4.1, v4.2)                         │
│                                                                       │
│ Best for:                                                            │
│ ├── Migrating Linux/ZFS workloads to AWS                         │
│ ├── Applications needing ZFS features (snaps, compression)      │
│ ├── Dev/test with instant clones                                  │
│ └── Database testing (clone prod DB snapshot instantly)           │
│                                                                       │
│ vs EFS:                                                              │
│ ├── EFS: Multi-AZ, elastic, shared, general purpose             │
│ ├── OpenZFS: Single-AZ, fixed size, ZFS features, higher perf  │
│ └── Choose EFS for most shared NFS needs; OpenZFS for ZFS needs │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: AWS Storage Gateway

```
┌─────────────────────────────────────────────────────────────────────┐
│           AWS STORAGE GATEWAY                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Hybrid cloud storage service. Connects on-premises apps    │
│ to AWS storage using standard protocols (NFS, SMB, iSCSI, VTL). │
│                                                                       │
│ How it works:                                                        │
│ ┌───────────────────┐     ┌───────────────────┐                    │
│ │ On-Premises       │     │ AWS Cloud          │                    │
│ │                   │     │                    │                    │
│ │ ┌───────────────┐ │     │ ┌──────────────┐  │                    │
│ │ │ Applications  │ │     │ │ S3 / FSx /   │  │                    │
│ │ │ (NFS/SMB/     │ │────►│ │ EBS / Glacier│  │                    │
│ │ │  iSCSI/VTL)   │ │     │ └──────────────┘  │                    │
│ │ └───────┬───────┘ │     │                    │                    │
│ │         │         │     │                    │                    │
│ │ ┌───────┴───────┐ │     │                    │                    │
│ │ │ Storage       │ │     │                    │                    │
│ │ │ Gateway VM    │ │     │                    │                    │
│ │ │ (local cache) │ │     │                    │                    │
│ │ └───────────────┘ │     │                    │                    │
│ └───────────────────┘     └───────────────────┘                    │
│                                                                       │
│ Gateway types:                                                       │
│ ┌────────────────┬──────────────┬──────────────┬─────────────────┐ │
│ │ Type           │ Protocol     │ Backend      │ Use Case         │ │
│ ├────────────────┼──────────────┼──────────────┼─────────────────┤ │
│ │ S3 File        │ NFS / SMB    │ S3           │ File shares     │ │
│ │ Gateway        │              │              │ backed by S3    │ │
│ │                │              │              │                  │ │
│ │ FSx File       │ SMB          │ FSx Windows  │ Windows file    │ │
│ │ Gateway        │              │              │ shares + cache  │ │
│ │                │              │              │                  │ │
│ │ Volume         │ iSCSI        │ S3 + EBS     │ Block storage   │ │
│ │ Gateway        │              │              │ with cloud backup│ │
│ │                │              │              │                  │ │
│ │ Tape           │ VTL          │ S3 Glacier   │ Backup to       │ │
│ │ Gateway        │ (iSCSI)     │              │ virtual tapes   │ │
│ └────────────────┴──────────────┴──────────────┴─────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Storage Gateway Types (Full Portal Walkthrough)

```
Console → Storage Gateway → Create gateway

┌─────────────────────────────────────────────────────────────────┐
│           CREATE GATEWAY                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Gateway settings ──                                          │
│                                                                   │
│ Gateway name: [onprem-file-gateway]                            │
│                                                                   │
│ Gateway type:                                                   │
│   ● Amazon S3 File Gateway                                     │
│   ○ Amazon FSx File Gateway                                    │
│   ○ Volume Gateway                                              │
│   ○ Tape Gateway                                                │
│                                                                   │
│ ── S3 File Gateway ──                                           │
│ → Files stored locally appear as NFS/SMB shares               │
│ → Data automatically synced to S3                              │
│ → Local cache for frequently accessed files                   │
│ → Each file = S3 object (1:1 mapping)                         │
│ → S3 lifecycle rules apply (transition to Glacier, etc.)      │
│                                                                   │
│ ── Host platform ──                                             │
│ ● Amazon EC2 (gateway in AWS)                                  │
│ ○ VMware ESXi                                                   │
│ ○ Microsoft Hyper-V                                             │
│ ○ Linux KVM                                                     │
│ ○ Hardware appliance                                            │
│                                                                   │
│ → EC2: Deploy gateway as EC2 instance in your VPC             │
│ → VMware/Hyper-V/KVM: Deploy on-premises as virtual machine  │
│ → Hardware: Physical AWS appliance shipped to your datacenter │
│                                                                   │
│ ── File share configuration (after gateway creation) ──        │
│                                                                   │
│ File share type: ● NFS ○ SMB                                  │
│ S3 bucket: [on-prem-backup-bucket ▼]                          │
│ S3 prefix: [file-gateway/]                                    │
│ Storage class: [S3 Standard ▼]                                │
│ IAM role: [file-gateway-s3-role ▼]                            │
│                                                                   │
│ Squash settings (NFS):                                         │
│   Root squash: [No root squash ▼]                             │
│   → No root squash: Root on client = root on gateway         │
│   → Root squash: Root mapped to nfsnobody                    │
│   → All squash: All users mapped to nfsnobody                │
│                                                                   │
│ Cache refresh: [5] minutes                                    │
│ → How often gateway checks S3 for changes                    │
│                                                                   │
│                    [Create gateway]                             │
└─────────────────────────────────────────────────────────────────┘

Volume Gateway modes:
├── Cached volumes: Primary data in S3, frequently accessed data cached locally
│   → Good when cloud is primary, need local cache for performance
├── Stored volumes: Full data locally, async backup to S3 as EBS snapshots
│   → Good when local access is critical, need cloud backup
└── Both present as iSCSI block devices to on-premises servers

Tape Gateway:
├── Virtual tape library (VTL) backed by S3 and Glacier
├── Compatible with: Veeam, Veritas, NetBackup, Commvault, etc.
├── Virtual tapes: 100 GiB - 5 TiB each
├── Total library: up to 1 PB
└── Tapes archived to: S3 Glacier or S3 Glacier Deep Archive
```

---

## Part 8: Terraform & CLI Examples

```hcl
# FSx for Windows File Server
resource "aws_fsx_windows_file_system" "corp" {
  storage_capacity    = 1024
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  throughput_capacity = 256
  deployment_type     = "MULTI_AZ_2"
  storage_type        = "SSD"

  self_managed_active_directory {
    dns_ips     = ["10.0.1.10", "10.0.2.10"]
    domain_name = "corp.example.com"
    username    = "FSxAdmin"
    password    = var.ad_password
  }

  security_group_ids        = [aws_security_group.fsx.id]
  automatic_backup_retention_days = 7

  tags = { Name = "corp-shared-drive" }
}

# FSx for Lustre with S3 data repository
resource "aws_fsx_lustre_file_system" "ml" {
  storage_capacity = 4800
  subnet_ids       = [aws_subnet.private_a.id]
  deployment_type  = "SCRATCH_2"

  import_path = "s3://${aws_s3_bucket.datasets.id}/training/"
  export_path = "s3://${aws_s3_bucket.datasets.id}/results/"

  security_group_ids = [aws_security_group.lustre.id]

  tags = { Name = "ml-training-data" }
}
```

```bash
# Create FSx for Windows
aws fsx create-file-system \
  --file-system-type WINDOWS \
  --storage-capacity 1024 \
  --storage-type SSD \
  --subnet-ids subnet-abc123 subnet-def456 \
  --windows-configuration '{
    "ThroughputCapacity": 256,
    "DeploymentType": "MULTI_AZ_2",
    "SelfManagedActiveDirectoryConfiguration": {
      "DomainName": "corp.example.com",
      "DnsIps": ["10.0.1.10"],
      "UserName": "FSxAdmin",
      "Password": "********"
    }
  }'

# Create FSx for Lustre
aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 4800 \
  --subnet-ids subnet-abc123 \
  --lustre-configuration '{
    "DeploymentType": "SCRATCH_2",
    "ImportPath": "s3://my-datasets/training/"
  }'
```

---

## Part 9: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Enterprise Windows File Shares                           │
│ ├── FSx for Windows (Multi-AZ) + AWS Managed AD                 │
│ ├── DFS Namespaces for scaling beyond 64 TB                     │
│ ├── Shadow Copies for user self-service restore                  │
│ ├── On-premises access via Direct Connect/VPN                   │
│ └── Daily automatic backups, 30-day retention                    │
│                                                                       │
│ Pattern 2: ML Training Pipeline                                     │
│ S3 (datasets) → FSx Lustre (training) → EC2/SageMaker (GPU)    │
│ ├── Lustre lazy-loads from S3 (no data copy needed)             │
│ ├── Scratch deployment for short training jobs                   │
│ ├── Results exported back to S3                                   │
│ └── Delete Lustre after training (data safe in S3)              │
│                                                                       │
│ Pattern 3: Hybrid Cloud with S3 File Gateway                       │
│ ├── On-premises apps write to NFS/SMB share                     │
│ ├── Data automatically stored in S3                              │
│ ├── Local cache for low-latency access                           │
│ ├── S3 lifecycle rules for cost optimization                    │
│ └── Cloud-side processing (Lambda, Athena) on S3 data           │
│                                                                       │
│ Pattern 4: Tape Backup Migration                                    │
│ ├── Replace physical tape library with Tape Gateway             │
│ ├── Existing backup software (Veeam) writes to virtual tapes   │
│ ├── Tapes stored in S3 Glacier/Deep Archive                     │
│ └── 90% cost reduction vs physical tape management              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
FSx & Storage Gateway Quick Reference:
├── FSx Windows: SMB, AD integration, Multi-AZ, Windows apps
├── FSx Lustre: HPC, ML, S3 integration, sub-ms latency
├── FSx NetApp: Multi-protocol (NFS+SMB+iSCSI), enterprise features
├── FSx OpenZFS: NFS, ZFS features (snapshots, compression)
├── S3 File Gateway: On-prem NFS/SMB → S3
├── FSx File Gateway: On-prem SMB → FSx Windows (with cache)
├── Volume Gateway: On-prem iSCSI → S3/EBS snapshots
├── Tape Gateway: On-prem VTL → S3 Glacier
└── ⚡ Most common: EFS (shared Linux), FSx Windows (shared Windows)
```

---

## What's Next?

In **Chapter 25: RDS - Relational Database Service**, we'll cover managed relational databases — engine types, Multi-AZ, read replicas, backups, and full portal walkthrough.
