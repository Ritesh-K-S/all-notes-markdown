# Chapter 23: EFS - Elastic File System

---

## Table of Contents

- [Overview](#overview)
- [Part 1: EFS Fundamentals](#part-1-efs-fundamentals)
- [Part 2: Creating an EFS File System (Full Portal Walkthrough)](#part-2-creating-an-efs-file-system-full-portal-walkthrough)
- [Part 3: Mount Targets & Networking](#part-3-mount-targets--networking)
- [Part 4: Access Points](#part-4-access-points)
- [Part 5: Performance Modes & Throughput Modes](#part-5-performance-modes--throughput-modes)
- [Part 6: Lifecycle Management & Storage Classes](#part-6-lifecycle-management--storage-classes)
- [Part 7: Security & Encryption](#part-7-security--encryption)
- [Part 8: Mounting on EC2](#part-8-mounting-on-ec2)
- [Part 9: Terraform & CLI Examples](#part-9-terraform--cli-examples)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is EFS? Why Do We Need It?

Imagine you have 3 web servers behind a load balancer, and all of them need to access the same uploaded files. With EBS, each server has its **own separate disk** — if a user uploads a photo to Server 1, Servers 2 and 3 can't see it. 

**EFS solves this problem.** It's like a **shared network drive** (similar to a shared folder on your office network) that all your servers can access simultaneously. When any server writes a file, all other servers can immediately read it.

**Think of it this way:**
- **EBS** = Your personal USB drive (one computer at a time)
- **EFS** = A shared Google Drive folder (everyone accesses the same files)
- **S3** = A cloud storage locker (great for files, but not a file system you can "mount")

**Simple real-world examples:**
- 🌐 A WordPress site with 5 EC2 instances sharing the same `wp-content/uploads` folder
- 🏠 A home directory server where each developer's files are available on any instance they log into
- 🤖 Machine learning training jobs that need to read the same large dataset from multiple instances

> ⚠️ **EFS is Linux-only** (NFS protocol). If you need shared storage for Windows servers, use **FSx for Windows File Server** (Chapter 24).

Amazon EFS is a fully managed, elastic, shared file system that can be mounted by multiple EC2 instances simultaneously. It uses the NFS protocol and automatically grows and shrinks as you add/remove files.

```
What you'll learn:
├── EFS Fundamentals (NFS, shared access, elastic)
├── Creating an EFS File System (Full Portal Walkthrough)
│   ├── File system settings
│   ├── Network (mount targets, subnets, security groups)
│   └── File system policy
├── Mount Targets (per-AZ ENIs)
├── Access Points (application-specific entry points)
├── Performance Modes (General Purpose vs Max I/O)
├── Throughput Modes (Elastic, Bursting, Provisioned)
├── Storage Classes (Standard, IA, Archive)
├── Lifecycle Management (automatic tiering)
├── Security & Encryption (at rest + in transit)
├── Mounting on EC2 (amazon-efs-utils, fstab)
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: EFS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           EFS CORE CONCEPTS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is EFS?                                                         │
│ ├── Managed NFS (Network File System) v4.1                        │
│ ├── Shared file system — multiple EC2 instances mount same FS    │
│ ├── Multi-AZ: Accessible from any AZ in the region               │
│ ├── Elastic: Grows/shrinks automatically (no provisioning)       │
│ ├── Pay only for storage used (no pre-allocation)                │
│ └── POSIX-compliant (Linux only — no Windows support)            │
│                                                                       │
│ Architecture:                                                        │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │                         VPC                                    │  │
│ │                                                                │  │
│ │  AZ-a                AZ-b                AZ-c                 │  │
│ │  ┌────────────┐    ┌────────────┐    ┌────────────┐          │  │
│ │  │ Subnet     │    │ Subnet     │    │ Subnet     │          │  │
│ │  │ ┌────────┐ │    │ ┌────────┐ │    │ ┌────────┐ │          │  │
│ │  │ │ EC2    │ │    │ │ EC2    │ │    │ │ EC2    │ │          │  │
│ │  │ └───┬────┘ │    │ └───┬────┘ │    │ └───┬────┘ │          │  │
│ │  │     │      │    │     │      │    │     │      │          │  │
│ │  │ ┌───┴────┐ │    │ ┌───┴────┐ │    │ ┌───┴────┐ │          │  │
│ │  │ │ Mount  │ │    │ │ Mount  │ │    │ │ Mount  │ │          │  │
│ │  │ │ Target │ │    │ │ Target │ │    │ │ Target │ │          │  │
│ │  │ └───┬────┘ │    │ └───┬────┘ │    │ └───┬────┘ │          │  │
│ │  └─────┼──────┘    └─────┼──────┘    └─────┼──────┘          │  │
│ │        └─────────────────┼─────────────────┘                  │  │
│ │                    ┌─────┴─────┐                               │  │
│ │                    │  EFS File │                               │  │
│ │                    │  System   │                               │  │
│ │                    └───────────┘                               │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ EFS vs EBS vs S3:                                                   │
│ ┌──────────────┬─────────────────┬────────────┬──────────────────┐ │
│ │              │ EFS             │ EBS         │ S3               │ │
│ ├──────────────┼─────────────────┼────────────┼──────────────────┤ │
│ │ Protocol     │ NFS (POSIX)     │ Block      │ HTTP/S (API)     │ │
│ │ Shared       │ Yes (multi-AZ)  │ No (1 AZ)  │ Yes (region)     │ │
│ │ Auto-scale   │ Yes             │ No (fixed) │ Yes              │ │
│ │ OS Support   │ Linux only      │ Linux/Win  │ Any              │ │
│ │ Latency      │ Low (ms)        │ Sub-ms     │ Varies           │ │
│ │ Cost/GB      │ $0.30 Standard  │ $0.08 gp3  │ $0.023           │ │
│ │ Use case     │ Shared files,   │ Boot disk, │ Objects,         │ │
│ │              │ CMS, home dirs  │ databases  │ backups          │ │
│ └──────────────┴─────────────────┴────────────┴──────────────────┘ │
│                                                                       │
│ Pricing (us-east-1):                                                │
│ ├── Standard: $0.30/GB/month                                     │
│ ├── Infrequent Access (IA): $0.016/GB/month + $0.01/GB access   │
│ ├── Archive: $0.008/GB/month + access fees                       │
│ ├── One Zone Standard: $0.16/GB/month (single AZ, 47% cheaper) │
│ └── Provisioned throughput: $6.00/MB/s/month above baseline     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating an EFS File System (Full Portal Walkthrough)

```
Console → EFS → Create file system → Customize

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: FILE SYSTEM SETTINGS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [shared-app-data]                                        │
│ → Descriptive name for identification                          │
│                                                                   │
│ Storage class:                                                  │
│   ● Regional (Multi-AZ)                                        │
│   ○ One Zone                                                    │
│                                                                   │
│ → Regional: Data stored across 3+ AZs. Higher availability    │
│   and durability. Use for production workloads.                │
│ → One Zone: Single AZ. 47% cheaper. Use for dev/test or       │
│   data that can be recreated. Specify AZ if selected.         │
│     Availability Zone: [us-east-1a ▼]                          │
│                                                                   │
│ ── Automatic backups ──                                        │
│ ☑ Enable automatic backups                                    │
│ → Uses AWS Backup. Daily backups with 35-day retention.       │
│ → ⚡ Enable for production file systems                       │
│                                                                   │
│ ── Lifecycle management ──                                     │
│ Transition into IA:                                            │
│   [30 days since last access ▼]                                │
│   Options: 1, 7, 14, 30, 60, 90 days, or None                │
│ → Automatically moves files not accessed in X days to IA     │
│ → IA storage: $0.016/GB vs $0.30/GB (94% cheaper!)          │
│                                                                   │
│ Transition into Archive:                                       │
│   [90 days since last access ▼]                                │
│ → Even cheaper than IA for rarely accessed files              │
│                                                                   │
│ Transition out of IA:                                          │
│   [On first access ▼]                                          │
│   Options: On first access, None                               │
│ → Move back to Standard when accessed                         │
│ → "None" = stays in IA even when accessed                    │
│                                                                   │
│ ── Encryption ──                                                │
│ ☑ Enable encryption of data at rest                           │
│ KMS key: [aws/elasticfilesystem (default) ▼]                  │
│ → ⚡ Always enable encryption                                 │
│                                                                   │
│ ── Performance settings ──                                     │
│                                                                   │
│ Throughput mode:                                                │
│   ● Elastic (recommended)                                      │
│   ○ Bursting                                                    │
│   ○ Provisioned                                                │
│                                                                   │
│ → Elastic: Automatically scales throughput up/down.           │
│   Pay per-use. Best for most workloads. Up to 10 GB/s reads. │
│ → Bursting: Throughput scales with storage size.              │
│   Baseline: 50 KB/s per GB. Burst: up to 100 MB/s.          │
│   ⚠️ Small file systems = low throughput.                     │
│ → Provisioned: Fixed throughput regardless of size.           │
│   Specify: [256] MB/s. Use when you need guaranteed           │
│   throughput with small storage size.                          │
│                                                                   │
│ Performance mode:                                               │
│   ● General Purpose (recommended)                              │
│   ○ Max I/O                                                     │
│                                                                   │
│ → General Purpose: Lower latency. Best for most use cases.   │
│   Web serving, CMS, home directories, dev environments.       │
│ → Max I/O: Higher throughput and IOPS but higher latency.    │
│   Big data, media processing, highly parallelized workloads. │
│   ⚠️ Cannot be changed after creation.                       │
│                                                                   │
│ ── Tags ──                                                      │
│ Key: [environment]  Value: [production]                       │
│ Key: [team]         Value: [platform]                         │
│                                                                   │
│                    [Next]                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│           STEP 2: NETWORK ACCESS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ VPC: [vpc-0abc123 (main-vpc) ▼]                                │
│                                                                   │
│ Mount targets (one per AZ):                                    │
│ ┌────────────────┬──────────────────┬─────────────────────────┐│
│ │ Availability   │ Subnet           │ Security Groups         ││
│ │ Zone           │                  │                          ││
│ ├────────────────┼──────────────────┼─────────────────────────┤│
│ │ us-east-1a     │ subnet-priv-1a ▼│ sg-efs-access ▼        ││
│ │ us-east-1b     │ subnet-priv-1b ▼│ sg-efs-access ▼        ││
│ │ us-east-1c     │ subnet-priv-1c ▼│ sg-efs-access ▼        ││
│ └────────────────┴──────────────────┴─────────────────────────┘│
│                                                                   │
│ → Mount target = ENI in each AZ for NFS access               │
│ → Place in PRIVATE subnets                                    │
│ → Security group must allow NFS (port 2049) from EC2 SG     │
│ → ⚡ Create mount targets in ALL AZs where EC2 runs           │
│                                                                   │
│ Security group for EFS:                                        │
│   Inbound: TCP 2049 (NFS) from EC2 security group            │
│   Outbound: All traffic                                        │
│                                                                   │
│                    [Next]                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│           STEP 3: FILE SYSTEM POLICY (Optional)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ☑ Prevent root access by default                               │
│ ☑ Enforce read-only access by default                          │
│ ☑ Prevent anonymous access                                     │
│ ☑ Enforce in-transit encryption for all clients               │
│                                                                   │
│ → These create a resource-based policy for the file system    │
│ → "Enforce in-transit encryption" = require TLS mount         │
│ → ⚡ Enable all for production                                 │
│                                                                   │
│                    [Next → Review → Create]                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Mount Targets & Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│           MOUNT TARGETS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Each mount target:                                                   │
│ ├── Is an ENI (Elastic Network Interface) in a subnet             │
│ ├── Gets a private IP address                                     │
│ ├── Gets a DNS name                                                │
│ ├── Has security group(s) attached                                │
│ └── One per AZ per file system                                    │
│                                                                       │
│ DNS resolution:                                                      │
│ ├── File system DNS: fs-abc123.efs.us-east-1.amazonaws.com       │
│ ├── Resolves to mount target IP in the same AZ as the client    │
│ ├── Automatic AZ-aware routing (no cross-AZ traffic)            │
│ └── Requires VPC DNS hostnames and resolution enabled            │
│                                                                       │
│ Cross-VPC access:                                                    │
│ ├── VPC Peering: Mount from peered VPC                           │
│ ├── Transit Gateway: Mount from connected VPCs                   │
│ └── Use mount target IP address directly (not DNS)               │
│                                                                       │
│ On-premises access:                                                  │
│ ├── Via AWS Direct Connect or VPN                                │
│ ├── Mount using mount target IP address                          │
│ └── ⚡ Consider latency for on-premises NFS mounts               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Access Points

```
Console → EFS → File system → Access points → Create access point

┌─────────────────────────────────────────────────────────────────┐
│           CREATE ACCESS POINT                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [app-data-access]                                        │
│                                                                   │
│ Root directory path: [/app/data]                               │
│ → The directory in EFS that this access point exposes         │
│ → Clients mounting via this AP see this as their root         │
│ → Created automatically if it doesn't exist                   │
│                                                                   │
│ ── POSIX user ──                                                │
│ User ID: [1000]                                                │
│ Group ID: [1000]                                               │
│ Secondary group IDs: [1001, 1002] (optional)                  │
│ → Override the NFS client's UID/GID                           │
│ → All file operations use this identity                       │
│ → Enforced regardless of what the client sends               │
│                                                                   │
│ ── Root directory creation permissions ──                      │
│ Owner user ID: [1000]                                         │
│ Owner group ID: [1000]                                        │
│ POSIX permissions: [755]                                      │
│ → Applied when EFS creates the root directory automatically  │
│                                                                   │
│ Why access points?                                             │
│ ├── Application isolation (each app gets its own "root")     │
│ ├── Enforce user identity (UID/GID)                          │
│ ├── Simplify permissions management                           │
│ ├── Use with Lambda (Lambda mounts EFS via access point)     │
│ └── IAM authorization for access control                      │
│                                                                   │
│                    [Create access point]                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Performance Modes & Throughput Modes

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERFORMANCE & THROUGHPUT                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Performance Modes:                                                   │
│ ┌──────────────────┬──────────────────┬───────────────────────────┐│
│ │                  │ General Purpose   │ Max I/O                   ││
│ ├──────────────────┼──────────────────┼───────────────────────────┤│
│ │ Latency          │ Low (sub-ms)      │ Higher (slightly)         ││
│ │ IOPS             │ Up to 35,000      │ 500,000+                  ││
│ │ Throughput       │ Up to 10 GB/s     │ Up to 10 GB/s             ││
│ │ Best for         │ Web, CMS, home    │ Big data, HPC, media      ││
│ │                  │ dirs, most apps   │ processing                ││
│ │ ⚠️ Change after  │ Yes               │ Yes                       ││
│ │ creation?        │                   │                           ││
│ └──────────────────┴──────────────────┴───────────────────────────┘│
│                                                                       │
│ Throughput Modes:                                                    │
│ ┌──────────────────┬──────────────────┬───────────────────────────┐│
│ │                  │ Elastic           │ Bursting     │ Provisioned││
│ ├──────────────────┼──────────────────┼──────────────┼────────────┤│
│ │ How it scales    │ Auto up/down      │ With storage │ Fixed      ││
│ │ Baseline         │ N/A (elastic)     │ 50KB/s/GB    │ You set    ││
│ │ Max read         │ 10 GB/s           │ 100 MB/s     │ 1 GB/s     ││
│ │ Max write        │ 3 GB/s            │ 100 MB/s     │ 1 GB/s     ││
│ │ Cost             │ Per-use           │ Included     │ Fixed/mo   ││
│ │ Best for         │ Most workloads    │ Large FS,    │ Small FS,  ││
│ │                  │ (unpredictable)   │ steady load  │ high thru  ││
│ └──────────────────┴──────────────────┴──────────────┴────────────┘│
│                                                                       │
│ ⚡ Recommendation: Start with Elastic throughput mode.               │
│ Switch to Provisioned only if you have a small FS needing high     │
│ sustained throughput.                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Lifecycle Management & Storage Classes

```
┌─────────────────────────────────────────────────────────────────────┐
│           EFS STORAGE CLASSES & LIFECYCLE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Storage classes:                                                     │
│ ├── Standard: Frequently accessed. $0.30/GB/month.               │
│ ├── Infrequent Access (IA): $0.016/GB/month + $0.01/GB access.  │
│ ├── Archive: $0.008/GB/month + access fees. Rarely accessed.    │
│ └── One Zone variants: Same tiers, single AZ, ~47% cheaper.     │
│                                                                       │
│ Lifecycle policies:                                                  │
│ ├── Transition into IA: After 1/7/14/30/60/90 days             │
│ ├── Transition into Archive: After 90/180/270/365 days          │
│ ├── Transition out of IA: On first access (or never)            │
│ └── Automatic, per-file (not per-file-system)                   │
│                                                                       │
│ ⚡ Cost savings example:                                             │
│ 1 TB EFS, 80% files rarely accessed:                               │
│ All Standard: 1,000 GB × $0.30 = $300/month                      │
│ With IA:      200 GB × $0.30 + 800 GB × $0.016 = $72.80/month   │
│ Savings: 76%!                                                       │
│                                                                       │
│ Configure: EFS → File system → Edit → Lifecycle management        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Security & Encryption

```
┌─────────────────────────────────────────────────────────────────────┐
│           EFS SECURITY                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Security layers:                                                     │
│                                                                       │
│ 1. Network: Security groups on mount targets                       │
│    ├── Allow TCP 2049 (NFS) from EC2 security group              │
│    └── Standard SG inbound/outbound rules                         │
│                                                                       │
│ 2. IAM: Resource-based policy on file system                       │
│    ├── Control which IAM entities can mount                       │
│    ├── Enforce encryption in transit                               │
│    └── Prevent root access                                         │
│                                                                       │
│ 3. POSIX permissions: Standard Linux file permissions              │
│    ├── UID/GID-based access control                               │
│    └── Enforced via access points                                 │
│                                                                       │
│ Encryption at rest:                                                  │
│ ├── AES-256 via AWS KMS                                           │
│ ├── Enable at creation time (cannot add later!)                  │
│ ├── Transparent to clients                                        │
│ └── AWS-managed key (free) or customer-managed key               │
│                                                                       │
│ Encryption in transit:                                               │
│ ├── TLS 1.2 via amazon-efs-utils mount helper                   │
│ ├── Mount with: sudo mount -t efs -o tls fs-abc123:/ /efs       │
│ ├── Can be enforced via file system policy                       │
│ └── ⚡ Always use TLS for production                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Mounting on EC2

```
# Step 1: Install amazon-efs-utils
sudo yum install -y amazon-efs-utils        # Amazon Linux
sudo apt install -y amazon-efs-utils        # Ubuntu

# Step 2: Create mount point
sudo mkdir /efs

# Step 3: Mount using EFS mount helper (recommended)
# Without TLS:
sudo mount -t efs fs-abc123def:/ /efs

# With TLS (encryption in transit):
sudo mount -t efs -o tls fs-abc123def:/ /efs

# With access point:
sudo mount -t efs -o tls,accesspoint=fsap-abc123 fs-abc123def:/ /efs

# Step 4: Verify
df -h /efs
# Shows: 8.0E (8 Exabytes — appears unlimited)

# Step 5: Auto-mount on reboot (add to /etc/fstab)
# Using EFS mount helper:
fs-abc123def:/ /efs efs _netdev,tls 0 0

# Using NFS (without efs-utils):
fs-abc123def.efs.us-east-1.amazonaws.com:/ /efs nfs4 \
  nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0

# Troubleshooting mount issues:
# 1. Check security group allows TCP 2049
# 2. Check mount target exists in the instance's AZ
# 3. Check VPC DNS resolution is enabled
# 4. Check NFS client is installed: rpm -qa | grep nfs-utils
# 5. Check EFS file system policy doesn't deny access
```

---

## Part 9: Terraform & CLI Examples

```hcl
# EFS File System
resource "aws_efs_file_system" "shared" {
  creation_token = "shared-app-data"
  encrypted      = true

  performance_mode = "generalPurpose"
  throughput_mode  = "elastic"

  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }

  lifecycle_policy {
    transition_to_archive = "AFTER_90_DAYS"
  }

  tags = {
    Name        = "shared-app-data"
    Environment = "prod"
  }
}

# Mount targets (one per AZ)
resource "aws_efs_mount_target" "az_a" {
  file_system_id  = aws_efs_file_system.shared.id
  subnet_id       = aws_subnet.private_a.id
  security_groups = [aws_security_group.efs.id]
}

resource "aws_efs_mount_target" "az_b" {
  file_system_id  = aws_efs_file_system.shared.id
  subnet_id       = aws_subnet.private_b.id
  security_groups = [aws_security_group.efs.id]
}

# Access point
resource "aws_efs_access_point" "app" {
  file_system_id = aws_efs_file_system.shared.id

  posix_user {
    uid = 1000
    gid = 1000
  }

  root_directory {
    path = "/app/data"
    creation_info {
      owner_uid   = 1000
      owner_gid   = 1000
      permissions = "755"
    }
  }

  tags = { Name = "app-data-access" }
}

# Security group for EFS
resource "aws_security_group" "efs" {
  name_prefix = "efs-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 2049
    to_port         = 2049
    protocol        = "tcp"
    security_groups = [aws_security_group.ec2.id]
  }
}
```

```bash
# Create file system
aws efs create-file-system \
  --creation-token my-efs \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --tags Key=Name,Value=shared-app-data

# Create mount target
aws efs create-mount-target \
  --file-system-id fs-abc123 \
  --subnet-id subnet-0abc123 \
  --security-groups sg-0def456

# Create access point
aws efs create-access-point \
  --file-system-id fs-abc123 \
  --posix-user Uid=1000,Gid=1000 \
  --root-directory Path=/app/data,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=755}

# Describe file systems
aws efs describe-file-systems --query "FileSystems[*].{Id:FileSystemId,Name:Name,Size:SizeInBytes.Value}"
```

---

## Part 10: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD EFS PATTERNS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Shared Web Content (WordPress/CMS)                       │
│ ├── Multiple EC2 instances behind ALB                             │
│ ├── EFS mounted at /var/www/html on all instances               │
│ ├── Upload once → visible on all servers                         │
│ ├── Lifecycle: IA after 30 days for media files                  │
│ └── Access points per application                                 │
│                                                                       │
│ Pattern 2: Lambda + EFS                                             │
│ ├── Lambda function mounts EFS via access point                  │
│ ├── Shared state/data between Lambda invocations                │
│ ├── ML model files loaded from EFS                               │
│ ├── Large file processing (>512 MB /tmp limit)                  │
│ └── Lambda must be in VPC with mount target                      │
│                                                                       │
│ Pattern 3: Container Shared Storage (ECS/EKS)                       │
│ ├── EFS as persistent volume for containers                      │
│ ├── Shared across tasks/pods                                      │
│ ├── Survives container restarts                                   │
│ └── Fargate: Native EFS volume support                           │
│                                                                       │
│ Pattern 4: Home Directories                                         │
│ ├── EFS mounted at /home on all instances                        │
│ ├── Access points per user (enforce UID/GID)                    │
│ ├── Lifecycle: IA after 90 days                                  │
│ └── Backup via AWS Backup                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
EFS Quick Reference:
├── Protocol: NFS v4.1 (Linux only)
├── Shared: Multi-AZ, multiple instances
├── Elastic: Auto grows/shrinks, no provisioning
├── Storage classes: Standard, IA, Archive (+ One Zone variants)
├── Performance modes: General Purpose (default), Max I/O
├── Throughput modes: Elastic (default), Bursting, Provisioned
├── Encryption: At rest (KMS) + in transit (TLS)
├── Mount targets: One ENI per AZ (TCP 2049)
├── Access points: Application-specific entry points
├── Lifecycle: Auto-tier to IA/Archive for cost savings
└── ⚡ Use EFS when multiple instances need shared file access
```

---

## What's Next?

In **Chapter 24: FSx & Storage Gateway**, we'll cover FSx for Windows File Server, FSx for Lustre, FSx for NetApp ONTAP, and AWS Storage Gateway for hybrid storage scenarios.
