# Chapter 58: AWS Snow Family & DataSync

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Snow Family Overview](#part-1-snow-family-overview)
- [Part 2: AWS Snowball Edge](#part-2-aws-snowball-edge)
- [Part 3: AWS Snowcone & Snowmobile](#part-3-aws-snowcone--snowmobile)
- [Part 4: Portal Walkthrough — Snow Family](#part-4-portal-walkthrough--snow-family)
- [Part 5: AWS DataSync](#part-5-aws-datasync)
- [Part 6: AWS Transfer Family](#part-6-aws-transfer-family)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### How Do I Move Lots of Data to AWS?

Imagine you need to move 100 TB of data to S3. Over a typical 1 Gbps internet connection, that would take **over 12 days** of continuous upload — and that's if nothing goes wrong.

**Sometimes it's faster to ship a hard drive via FedEx than to upload over the internet.** That's literally what the Snow Family is — AWS ships you a physical device, you load your data onto it, and ship it back.

**Which data transfer method should I use?**

| Data Size | Best Option | How It Works |
|-----------|------------|-------------|
| < 10 TB | **DataSync** (online) | Agent-based transfer over network |
| 10-80 TB | **Snowcone/Snowball** | Ship a physical device |
| 80-500 TB | **Snowball Edge** | Ship a larger device |
| Petabytes | **Multiple Snowballs** | Ship several devices |
| 10-100 PB | **Snowmobile** | AWS drives a truck to you (literally!) |
| Recurring sync | **DataSync** (scheduled) | Ongoing sync over network |
| Partner SFTP | **Transfer Family** | Managed SFTP/FTP server |

AWS provides both offline (Snow Family) and online (DataSync, Transfer Family) data transfer services for moving large volumes of data to and from AWS.

```
What you'll learn:
├── Snow Family: Offline data transfer devices
│   ├── Snowcone (8-14 TB)
│   ├── Snowball Edge (80-210 TB)
│   └── Snowmobile (100 PB)
├── AWS DataSync: Online data transfer
├── AWS Transfer Family: SFTP/FTPS/FTP to S3/EFS
└── When to use which service
```

---

## Part 1: Snow Family Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           SNOW FAMILY — WHEN TO USE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Rule of thumb: If transfer over network takes > 1 week → Snow   │
│                                                                       │
│ Data Size   │ Recommended           │ Transfer Time (1Gbps)      │
│ ────────────┼───────────────────────┼────────────────────────────│
│ < 10 TB     │ Network (DataSync)    │ ~1 day                     │
│ 10-80 TB    │ Snowcone / Snowball   │ 1-10 days (network)       │
│ 80-210 TB   │ Snowball Edge         │ Weeks (network)            │
│ 500 TB+     │ Multiple Snowballs    │ Months (network)           │
│ 10-100 PB   │ Snowmobile            │ Years (network)            │
│                                                                       │
│ Snow devices ship to you → load data → ship back → AWS uploads │
│                                                                       │
│ Additional use cases (not just migration):                         │
│ ├── Edge computing (run EC2/Lambda at remote locations)        │
│ ├── Data collection at edge locations                            │
│ └── Tactical edge (military, disaster recovery)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: AWS Snowball Edge

```
┌─────────────────────────────────────────────────────────────────────┐
│           SNOWBALL EDGE                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two variants:                                                        │
│                                                                       │
│ Storage Optimized:                                                   │
│ ├── 80 TB HDD usable storage                                     │
│ ├── 1 TB SSD (block volume)                                      │
│ ├── 40 vCPUs, 80 GB RAM                                          │
│ ├── S3-compatible object storage                                │
│ ├── EC2 compute (sbe-g, sbe-c instances)                       │
│ └── Best for: Large data migration + light compute             │
│                                                                       │
│ Compute Optimized:                                                   │
│ ├── 28 TB NVMe SSD usable storage                               │
│ ├── 104 vCPUs, 416 GB RAM                                       │
│ ├── Optional GPU (for ML inference at edge)                    │
│ ├── S3-compatible + EC2 compute + Lambda                       │
│ └── Best for: Edge computing, ML inference                     │
│                                                                       │
│ Features (both):                                                     │
│ ├── Cluster mode: 5-10 devices for increased storage/compute  │
│ ├── NFS mount: Mount as network file share                     │
│ ├── OpsHub: GUI management application                         │
│ ├── Encryption: 256-bit AES, keys managed by KMS             │
│ ├── Tamper-resistant, tamper-evident enclosure                 │
│ ├── E-Ink shipping label (auto-updates return address)       │
│ └── Trusted Platform Module (TPM)                              │
│                                                                       │
│ Pricing:                                                             │
│ ├── Service fee: Per job ($300 for 10-day use)                │
│ ├── Shipping: Standard rates                                    │
│ ├── Extra days: $30/day (Storage Opt), $50/day (Compute Opt) │
│ └── Data transfer: IN to S3 = free, OUT = standard rates     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: AWS Snowcone & Snowmobile

```
┌─────────────────────────────────────────────────────────────────────┐
│           SNOWCONE                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Smallest Snow device (portable: 4.5 lbs / 2.1 kg)                │
│                                                                       │
│ Snowcone:                                                            │
│ ├── 8 TB HDD usable storage                                      │
│ ├── 2 vCPUs, 4 GB RAM                                            │
│ ├── USB-C power, Wi-Fi optional                                 │
│ └── Can run DataSync agent (online transfer back to AWS)       │
│                                                                       │
│ Snowcone SSD:                                                        │
│ ├── 14 TB SSD usable storage                                     │
│ ├── Same compute as Snowcone                                    │
│ └── Faster I/O                                                   │
│                                                                       │
│ Use cases: IoT, edge data collection, content distribution      │
│ ⚡ Can ship back OR use DataSync over network                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           SNOWMOBILE                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 45-foot shipping container on a semi-trailer truck                │
│ ├── 100 PB per Snowmobile                                        │
│ ├── GPS tracking, 24/7 video surveillance                       │
│ ├── Escort security vehicle (optional)                           │
│ ├── 256-bit encryption                                           │
│ └── Temperature controlled, fire detection                      │
│                                                                       │
│ Use cases: Exabyte-scale migration (data center shutdown)       │
│ ⚠️ Available in select regions, requires AWS coordination       │
│ ⚡ Better than Snowball when transferring > 10 PB               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Portal Walkthrough — Snow Family

```
Console → AWS Snow Family → Create job

┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Job Type                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ● Import into Amazon S3                                        │
│ ○ Export from Amazon S3                                        │
│ ○ Local compute and storage only                               │
│                                                                   │
│                         [Next]                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Device & Storage                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Snow device:                                                    │
│ ○ Snowcone                    8 TB HDD                        │
│ ○ Snowcone SSD                14 TB SSD                       │
│ ● Snowball Edge Storage Opt.  80 TB                           │
│ ○ Snowball Edge Compute Opt.  28 TB SSD                       │
│                                                                   │
│ Storage type: ● Amazon S3  ○ NFS                              │
│                                                                   │
│ S3 bucket: [my-migration-bucket ▼]                            │
│ → Where data will be imported                                 │
│                                                                   │
│                         [Next]                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Compute (Optional)                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ EC2 AMIs: [Select AMIs to run on device ▼]                    │
│ → Pre-loaded AMIs for edge computing                          │
│                                                                   │
│ Lambda functions: [Select functions ▼]                        │
│ → Optional: Run Lambda at edge                                │
│                                                                   │
│                         [Next]                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Security & Notifications                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Encryption:                                                     │
│ KMS key: [aws/importexport (default) ▼]                      │
│ → Or choose custom KMS key                                    │
│                                                                   │
│ IAM role: [SnowballEdgeRole ▼]                               │
│ → Permissions for device to access S3, EC2                  │
│                                                                   │
│ SNS topic: [my-snow-notifications ▼]                         │
│ → Notifications for job status changes                       │
│                                                                   │
│                         [Next]                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Shipping Address                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [Data Center Ops]                                       │
│ Company: [My Company]                                         │
│ Street: [123 Data Center Rd]                                  │
│ City: [Seattle]  State: [WA]  ZIP: [98101]                   │
│ Country: [US]                                                  │
│ Phone: [+1-555-0123]                                          │
│                                                                   │
│ Shipping speed:                                                │
│ ● Two-day shipping                                             │
│ ○ One-day shipping (additional cost)                          │
│                                                                   │
│                         [Create job]                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 5: AWS DataSync

```
Console → AWS DataSync

┌─────────────────────────────────────────────────────────────────────┐
│           AWS DATASYNC                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Online data transfer service (automated, fast, secure)      │
│ Speed: Up to 10 Gbps (10x faster than open-source tools)        │
│                                                                       │
│ Supported locations:                                                 │
│ ├── Source: NFS, SMB, HDFS, self-managed object storage         │
│ │   Also: S3, EFS, FSx (AWS-to-AWS transfers)                  │
│ ├── Target: S3 (all classes), EFS, FSx (all types)             │
│ └── Other clouds: Azure Blob, Google Cloud Storage              │
│                                                                       │
│ Architecture (on-prem to AWS):                                     │
│ ┌──────────┐     ┌──────────────┐     ┌──────────────┐          │
│ │ On-Prem   │ ──→ │ DataSync     │ ──→ │ S3 / EFS /   │          │
│ │ NFS/SMB   │     │ Agent (VM)   │     │ FSx           │          │
│ └──────────┘     └──────────────┘     └──────────────┘          │
│                   Deployed on-prem       AWS target                │
│                   as VMware/Hyper-V/KVM                             │
│                                                                       │
│ Setup Walkthrough:                                                   │
│ Console → DataSync → Create agent                                  │
│ ├── Deploy agent VM on-premises (download OVA/VHD/AMI)         │
│ ├── Activate agent (enter VM IP in console)                      │
│ ├── Create source location: [NFS server: IP + export path]    │
│ ├── Create destination location: [S3 bucket or EFS]            │
│ └── Create task:                                                  │
│     ├── Source: [on-prem-nfs ▼]                                │
│     ├── Destination: [s3://migration-bucket ▼]                │
│     ├── Schedule: ○ Not scheduled ● Hourly/Daily/Custom      │
│     ├── Filtering:                                               │
│     │   ├── Include patterns: [*.csv, *.parquet]             │
│     │   └── Exclude patterns: [*.tmp, *.log]                 │
│     ├── Transfer mode:                                           │
│     │   ● Transfer all data                                    │
│     │   ○ Transfer only changed data                           │
│     ├── Verify data:                                             │
│     │   ● Full verification                                    │
│     │   ○ Verification on transfer only                        │
│     │   ○ None                                                  │
│     ├── Bandwidth limit: [0 (unlimited)] Mbps                 │
│     ├── Task logging: [CloudWatch log group ▼]               │
│     └── [Create task] → [Start task]                          │
│                                                                       │
│ ⚡ Automatic integrity validation (checksums)                     │
│ ⚡ Incremental transfers (only changed files)                    │
│ ⚡ Scheduling for recurring transfers                             │
│ ⚡ Free for agent; pay per GB transferred ($0.0125/GB)          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: AWS Transfer Family

```
Console → AWS Transfer Family

┌─────────────────────────────────────────────────────────────────────┐
│           AWS TRANSFER FAMILY                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed SFTP/FTPS/FTP/AS2 server for S3 or EFS             │
│ Use case: Partners/vendors who upload via FTP/SFTP                │
│                                                                       │
│ Supported protocols:                                                 │
│ ├── SFTP: SSH File Transfer Protocol (port 22)                 │
│ ├── FTPS: FTP over TLS (ports 21, 8192-8200)                 │
│ ├── FTP: Plain FTP (ports 21, 8192-8200) ⚠️ not encrypted    │
│ └── AS2: Applicability Statement 2 (B2B EDI)                  │
│                                                                       │
│ Setup:                                                               │
│ Console → Transfer Family → Create server                         │
│ ├── Protocol: ☑ SFTP ☐ FTPS ☐ FTP ☐ AS2                     │
│ ├── Identity provider:                                           │
│ │   ● Service managed (SSH keys stored in Transfer)           │
│ │   ○ AWS Directory Service (Active Directory)                │
│ │   ○ Custom (Lambda function for auth)                       │
│ │   ○ AWS IAM                                                   │
│ ├── Endpoint type:                                               │
│ │   ● Public (internet-facing)                                 │
│ │   ○ VPC (private, within VPC)                               │
│ │   ○ VPC with internet-facing (EIP)                          │
│ ├── Domain: ● S3 ○ EFS                                        │
│ ├── Logging: [CloudWatch log group ▼]                         │
│ └── [Create server]                                              │
│                                                                       │
│ Add user:                                                            │
│ ├── Username: [partner-user]                                     │
│ ├── IAM role: [TransferS3AccessRole] → S3 permissions          │
│ ├── Home directory: [my-bucket/partner-uploads/]              │
│ ├── SSH public key: [paste key]                                 │
│ └── Restricted: ☑ Chroot to home directory                     │
│                                                                       │
│ Endpoint: s-1234abcdef.server.transfer.us-east-1.amazonaws.com │
│ Client: sftp partner-user@s-1234abcdef.server.transfer...     │
│                                                                       │
│ Pricing: $0.30/hr (server) + $0.04/GB transferred             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & CLI Examples

```hcl
# DataSync - S3 location
resource "aws_datasync_location_s3" "destination" {
  s3_bucket_arn = aws_s3_bucket.migration.arn
  subdirectory  = "/data"

  s3_config {
    bucket_access_role_arn = aws_iam_role.datasync.arn
  }
}

# DataSync - Task
resource "aws_datasync_task" "migration" {
  name                     = "on-prem-to-s3"
  source_location_arn      = aws_datasync_location_nfs.source.arn
  destination_location_arn = aws_datasync_location_s3.destination.arn

  options {
    verify_mode   = "POINT_IN_TIME_CONSISTENT"
    posix_permissions = "NONE"
    uid           = "NONE"
    gid           = "NONE"
  }

  schedule {
    schedule_expression = "cron(0 2 * * ? *)"  # Daily at 2 AM
  }
}

# Transfer Family - SFTP Server
resource "aws_transfer_server" "sftp" {
  identity_provider_type = "SERVICE_MANAGED"
  endpoint_type          = "PUBLIC"
  protocols              = ["sftp"]

  logging_role = aws_iam_role.transfer_logging.arn
}

resource "aws_transfer_user" "partner" {
  server_id      = aws_transfer_server.sftp.id
  user_name      = "partner-user"
  role           = aws_iam_role.transfer_s3.arn
  home_directory = "/${aws_s3_bucket.uploads.id}/partner"
}
```

```bash
# Create Snowball job
aws snowball create-job \
  --job-type IMPORT \
  --resources '{"S3Resources":[{"BucketArn":"arn:aws:s3:::my-bucket"}]}' \
  --snowball-type EDGE \
  --shipping-option SECOND_DAY \
  --address-id ADID1234... \
  --kms-key-arn arn:aws:kms:... \
  --role-arn arn:aws:iam::123456789012:role/SnowballRole

# List Snow jobs
aws snowball list-jobs

# DataSync - start task
aws datasync start-task-execution \
  --task-arn arn:aws:datasync:...

# DataSync - list tasks
aws datasync list-tasks
```

---

## Part 8: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Data Center Migration (Petabytes)                       │
│ Phase 1: DataSync for incremental data (< 10 TB)                │
│ Phase 2: Snowball Edge devices for bulk data (100s TB)          │
│ Phase 3: DataSync for final delta sync                            │
│ → Hybrid approach minimizes total migration time                │
│                                                                       │
│ Pattern 2: Ongoing Data Sync                                        │
│ On-prem NFS → DataSync (scheduled hourly)                        │
│ → S3 → Processed by Lambda/Glue                                │
│ → Hybrid cloud, cloud-based analytics on on-prem data          │
│                                                                       │
│ Pattern 3: Secure File Exchange                                     │
│ Partners → Transfer Family (SFTP) → S3 bucket                   │
│ → S3 Event → Lambda → process/validate uploaded files          │
│ → SNS notification to internal team                              │
│ → Replace legacy FTP servers                                     │
│                                                                       │
│ Pattern 4: Edge Computing with Snow                                │
│ Remote site (oil rig, ship, disaster zone)                       │
│ → Snowball Edge Compute Opt. deployed                           │
│ → Run ML inference locally                                       │
│ → Collect data, ship back periodically                          │
│                                                                       │
│ ⚡ Decision guide:                                                    │
│ ├── Network transfer > 1 week? → Snow Family                   │
│ ├── Recurring transfers? → DataSync (scheduled)                │
│ ├── Partner SFTP/FTP? → Transfer Family                        │
│ ├── < 10 TB? → DataSync or S3 CLI/SDK                         │
│ └── Edge compute needed? → Snowball Edge Compute Opt.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Data Transfer Quick Reference:
├── Snow Family (offline):
│   ├── Snowcone: 8 TB HDD / 14 TB SSD (portable, 4.5 lbs)
│   ├── Snowball Edge Storage Opt: 80 TB (migration + light compute)
│   ├── Snowball Edge Compute Opt: 28 TB SSD (edge compute, GPU)
│   └── Snowmobile: 100 PB (truck, data center shutdown)
├── DataSync (online):
│   ├── Up to 10 Gbps, agent-based
│   ├── NFS/SMB/HDFS → S3/EFS/FSx
│   ├── Cross-cloud: Azure Blob, GCS
│   ├── Scheduling, filtering, verification
│   └── $0.0125/GB transferred
├── Transfer Family:
│   ├── Managed SFTP/FTPS/FTP/AS2 → S3/EFS
│   ├── $0.30/hr + $0.04/GB
│   └── Replace legacy FTP servers
├── ⚡ > 1 week over network → use Snow Family
├── ⚡ Recurring sync → DataSync with schedule
└── ⚡ Partner file upload → Transfer Family
```

---

## What's Next?

In **Chapter 59: AWS Well-Architected Framework**, we'll cover the 6 pillars (Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability), the Well-Architected Tool, and review process.
