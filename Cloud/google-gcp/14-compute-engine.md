# Chapter 14: Compute Engine

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Machine Types](#part-1-machine-types)
- [Part 2: Images](#part-2-images)
- [Part 3: Creating a VM Instance (Full Portal Walkthrough)](#part-3-creating-a-vm-instance-full-portal-walkthrough)
- [Part 4: SSH Access Methods](#part-4-ssh-access-methods)
- [Part 5: Persistent Disks](#part-5-persistent-disks)
- [Part 6: Spot VMs](#part-6-spot-vms)
- [Part 7: Sole-Tenant Nodes](#part-7-sole-tenant-nodes)
- [Part 8: Purchasing Discounts](#part-8-purchasing-discounts)
- [Part 9: Terraform Example](#part-9-terraform-example)
- [Part 10: gcloud CLI Reference](#part-10-gcloud-cli-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Part 12: Console Walkthrough: Managing & Deleting VMs](#part-12-console-walkthrough-managing--deleting-vms)
- [Part 13: Additional gcloud Commands](#part-13-additional-gcloud-commands)
- [Part 14: Ops Agent Installation](#part-14-ops-agent-installation)
- [Part 15: Troubleshooting Common Issues](#part-15-troubleshooting-common-issues)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Compute Engine is GCP's virtual machine service — the equivalent of AWS EC2 and Azure Virtual Machines. Every time you need a server (web server, database, build runner, ML workload), you create a Compute Engine VM instance. This chapter covers machine types, images, disks, full creation walkthrough, access methods, preemptible/spot VMs, sole-tenant nodes, and more.

```
What you'll learn:
├── Compute Engine Fundamentals
├── Machine Types (families, series, custom)
│   ├── General purpose (E2, N2, N2D, N4, C4A, T2D, T2A, Tau)
│   ├── Compute optimized (C2, C2D, C3, C3D, H3)
│   ├── Memory optimized (M1, M2, M3)
│   ├── Accelerator optimized (A2, A3, G2)
│   ├── Storage optimized (Z3)
│   └── Custom machine types
├── Images (OS)
│   ├── Public images
│   ├── Custom images
│   └── Machine images (full VM snapshot)
├── Creating a VM Instance (full portal walkthrough)
│   ├── Name, region, zone
│   ├── Machine configuration
│   ├── Boot disk (image, size, type)
│   ├── Networking (VPC, subnet, IP, firewall)
│   ├── Security (service account, shielded VM, SSH keys)
│   ├── Management (preemptible/spot, maintenance, labels)
│   └── Advanced (startup script, metadata)
├── SSH Access Methods
├── Disks (Persistent Disks & Local SSDs)
├── Snapshots
├── Preemptible & Spot VMs
├── Sole-Tenant Nodes
├── Terraform examples
├── gcloud CLI reference
└── Real-world patterns
```

---

## Part 1: Machine Types

```
┌─────────────────────────────────────────────────────────────────────┐
│           MACHINE TYPE NAMING                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Format: [series]-[type]-[cpus]                                     │
│                                                                       │
│ Example: n2-standard-8                                              │
│          │  │         └── 8 vCPUs                                   │
│          │  └── standard (balanced vCPU:memory ratio)              │
│          └── n2 (series/generation)                                 │
│                                                                       │
│ Types within a series:                                               │
│ ├── standard: Balanced (1 vCPU : 4 GB RAM)                       │
│ ├── highmem:  More memory (1 vCPU : 8 GB RAM)                    │
│ ├── highcpu:  More CPU (1 vCPU : 1 GB RAM)                       │
│ └── custom:   You choose exact vCPU + memory                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           MACHINE FAMILIES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┬───────────────┬──────────────┬──────────────────────┐ │
│ │ Series   │ Category      │ Processor    │ Use Case             │ │
│ ├──────────┼───────────────┼──────────────┼──────────────────────┤ │
│ │ E2       │ General       │ Intel/AMD    │ Dev/test, low-cost   │ │
│ │          │ (cost-opt.)   │ (shared/     │ web servers, small   │ │
│ │          │               │ dedicated)   │ apps, microservices  │ │
│ │          │               │              │ ⚡ Cheapest option    │ │
│ │          │               │              │                      │ │
│ │ N2/N2D   │ General       │ Intel (N2)   │ Most workloads.      │ │
│ │          │ (balanced)    │ AMD (N2D)    │ Web servers, app     │ │
│ │          │               │              │ servers, databases.  │ │
│ │          │               │              │ ⚡ Recommended default│ │
│ │          │               │              │                      │ │
│ │ N4       │ General       │ Intel (Emerald│ Latest gen, best    │ │
│ │          │ (latest gen)  │ Rapids)      │ price/performance    │ │
│ │          │               │              │                      │ │
│ │ C4A/T2A  │ General       │ Arm (Axion/  │ Web, containerized   │ │
│ │          │ (Arm-based)   │ Ampere)      │ workloads            │ │
│ │          │               │              │ ⚡ Best price/perf    │ │
│ │          │               │              │ (like AWS Graviton)  │ │
│ │          │               │              │                      │ │
│ │ C2/C3/   │ Compute       │ Intel/AMD    │ CPU-intensive:       │ │
│ │ C3D/H3   │ Optimized     │ (latest gen) │ batch, ML inference, │ │
│ │          │               │              │ gaming, HPC          │ │
│ │          │               │              │                      │ │
│ │ M1/M2/M3 │ Memory       │ Intel        │ In-memory DB, SAP    │ │
│ │          │ Optimized     │              │ HANA, big data       │ │
│ │          │               │              │ analytics. Up to     │ │
│ │          │               │              │ 30 TB RAM!           │ │
│ │          │               │              │                      │ │
│ │ A2/A3/G2 │ Accelerator  │ NVIDIA GPUs  │ ML training, video   │ │
│ │          │ (GPU)         │ (A100,H100,  │ encoding, rendering, │ │
│ │          │               │ L4)          │ LLM inference        │ │
│ │          │               │              │                      │ │
│ │ Z3       │ Storage       │ Intel + NVMe │ High-throughput      │ │
│ │          │ Optimized     │ local SSD    │ databases,           │ │
│ │          │               │              │ data analytics       │ │
│ └──────────┴───────────────┴──────────────┴──────────────────────┘ │
│                                                                       │
│ Sizes (example: n2-standard):                                       │
│ ┌──────────────────┬──────┬─────────┬──────────────────────────┐  │
│ │ Machine Type     │ vCPU │ RAM     │ Approx $/hr (on-demand) │  │
│ ├──────────────────┼──────┼─────────┼──────────────────────────┤  │
│ │ n2-standard-2    │ 2    │ 8 GB   │ $0.097                   │  │
│ │ n2-standard-4    │ 4    │ 16 GB  │ $0.194                   │  │
│ │ n2-standard-8    │ 8    │ 32 GB  │ $0.388                   │  │
│ │ n2-standard-16   │ 16   │ 64 GB  │ $0.776                   │  │
│ │ n2-standard-32   │ 32   │ 128 GB │ $1.553                   │  │
│ │ n2-standard-64   │ 64   │ 256 GB │ $3.106                   │  │
│ │ n2-standard-96   │ 96   │ 384 GB │ $4.659                   │  │
│ │ n2-standard-128  │ 128  │ 512 GB │ $6.213                   │  │
│ └──────────────────┴──────┴─────────┴──────────────────────────┘  │
│                                                                       │
│ ⚡ Decision guide:                                                    │
│ ├── "I don't know" → e2-medium (shared, cheap, good start)      │
│ ├── Production web → n2-standard-4 or c4a-standard-4 (Arm)     │
│ ├── CPU-heavy → c3-highcpu series                                │
│ ├── Memory-heavy → n2-highmem or m3 series                      │
│ ├── GPU/ML → a2-highgpu or g2-standard                          │
│ └── Try Arm (C4A/T2A) for best price/performance!               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Custom Machine Types

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM MACHINE TYPES                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCP unique feature: Pick EXACT vCPUs and memory you need.         │
│ (AWS and Azure don't offer this!)                                  │
│                                                                       │
│ Rules:                                                               │
│ ├── vCPUs: 1 or any even number (2, 4, 6, 8...) up to 96       │
│ ├── Memory: 0.9 GB to 6.5 GB per vCPU (standard range)         │
│ ├── Extended memory: Up to 8 GB per vCPU (additional charge)    │
│ └── Available for: N1, N2, N2D, E2 series                       │
│                                                                       │
│ Example: Your app needs 6 vCPUs and 24 GB RAM                     │
│ ├── n2-standard-8 (8 vCPU, 32 GB) → too much, overpaying      │
│ ├── n2-standard-4 (4 vCPU, 16 GB) → not enough                │
│ ├── Custom: n2-custom-6-24576 (6 vCPU, 24 GB) → perfect!      │
│ └── Save money by not rounding up to a predefined size          │
│                                                                       │
│ Create custom: During VM creation → Machine type → Custom        │
│   Cores: [6]                                                       │
│   Memory: [24] GB                                                  │
│   ☐ Extend memory (go beyond 6.5 GB/vCPU ratio)                 │
│                                                                       │
│ ⚡ Great for right-sizing after monitoring actual usage!            │
│    Monitor CPU/RAM → adjust custom type to actual needs.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           IMAGES                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Public images (Google-maintained):                                   │
│ ├── Debian 12 (default recommended)                               │
│ ├── Ubuntu 22.04/24.04 LTS                                        │
│ ├── CentOS Stream 9                                                │
│ ├── Rocky Linux 9                                                  │
│ ├── Red Hat Enterprise Linux 8/9 (paid)                           │
│ ├── SUSE Linux (paid)                                              │
│ ├── Windows Server 2019/2022 (paid)                               │
│ ├── Container-Optimized OS (COS) — for running containers       │
│ └── Deep Learning VM images (pre-installed TF, PyTorch)          │
│                                                                       │
│ Custom images:                                                       │
│ ├── Create from existing VM disk                                  │
│ ├── Import from on-premises (OVA, VMDK, VHD)                    │
│ ├── Build with Packer (automated image pipeline)                 │
│ ├── Share within project or across Org                            │
│ └── ⚡ Golden image pattern — pre-bake your app + configs        │
│                                                                       │
│ Machine images (full VM backup):                                    │
│ ├── Captures EVERYTHING: config, metadata, disks, permissions    │
│ ├── Can create new VM from machine image                         │
│ ├── Like an AWS AMI but also includes instance config            │
│ └── Use for: VM cloning, disaster recovery, migration           │
│                                                                       │
│ Image families:                                                      │
│ ├── Group of related images (e.g., my-web-app-prod)              │
│ ├── Family points to the LATEST image in the group               │
│ ├── Reference family name instead of specific image              │
│ ├── Upload new image to family → all new VMs use latest         │
│ └── Roll back by deprecating latest image                        │
│                                                                       │
│ Create custom image:                                                 │
│ Compute Engine → Images → Create image                             │
│   Name: [img-web-prod-v1-2-3]                                     │
│   Source: ● Disk (from existing VM disk)                          │
│   Source disk: [web-prod-01 boot disk ▼]                          │
│   Family: [img-web-prod]                                           │
│   Location: ● Multi-regional (more available)                    │
│             ○ Regional (cheaper, single region)                   │
│   Labels: version=1.2.3, app=web                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating a VM Instance (Full Portal Walkthrough)

```
Console → Compute Engine → VM instances → Create instance

┌─────────────────────────────────────────────────────────────────┐
│           CREATE AN INSTANCE                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Name ──                                                      │
│ Name: [web-prod-01]                                              │
│ ⚠️ Lowercase, hyphens only. Cannot be changed after creation!   │
│                                                                   │
│ ── Labels ──                                                    │
│ environment: prod                                               │
│ team: backend                                                   │
│ app: web-server                                                 │
│                                                                   │
│ ── Region and zone ──                                           │
│ Region: [asia-south1 ▼] (Mumbai)                               │
│ Zone: [asia-south1-a ▼]                                        │
│ ⚡ Zone is where the VM physically runs.                        │
│    For HA, spread VMs across zones (a, b, c).                  │
│                                                                   │
│ ── Machine configuration ──                                     │
│ Machine family:                                                 │
│   ● General purpose                                            │
│   ○ Compute optimized                                          │
│   ○ Memory optimized                                           │
│   ○ Accelerator (GPU)                                          │
│   ○ Storage optimized                                          │
│                                                                   │
│ Series: [N2 ▼] (or E2 for cost savings, C4A for Arm)         │
│                                                                   │
│ Machine type:                                                   │
│   ● Preset                                                     │
│     [n2-standard-4 ▼] (4 vCPU, 16 GB RAM)                   │
│   ○ Custom                                                     │
│     Cores: [6] Memory: [24] GB                                │
│     ☐ Extend memory                                            │
│                                                                   │
│ Display device: ☐ (enable for GPU passthrough/remote desktop) │
│                                                                   │
│ ── Confidential VM service ──                                   │
│ ☐ Enable Confidential VM service                              │
│   (Encrypts data in memory — AMD SEV-SNP)                     │
│   (For highly sensitive workloads — slight performance cost)  │
│                                                                   │
│ ── Boot disk ──                                                 │
│ [Change]                                                        │
│   ── Operating system ──                                       │
│   OS: [Debian ▼]                                               │
│   Version: [Debian GNU/Linux 12 (bookworm) ▼]                │
│   Or: [Ubuntu 24.04 LTS ▼]                                   │
│   Or: [Custom images ▼] → [img-web-prod ▼] (your image)     │
│                                                                   │
│   ── Boot disk type ──                                         │
│   Type: [Balanced persistent disk ▼] (pd-balanced = SSD)     │
│     ○ Standard persistent disk (pd-standard = HDD, cheapest) │
│     ● Balanced persistent disk (pd-balanced = SSD, default)  │
│     ○ SSD persistent disk (pd-ssd = fastest SSD)             │
│     ○ Extreme persistent disk (pd-extreme = highest IOPS)    │
│     ○ Hyperdisk Balanced (latest gen)                         │
│                                                                   │
│   Size (GB): [20] (min 10, max 65536)                         │
│                                                                   │
│   ☐ Delete boot disk when instance is deleted                 │
│     ● Checked (default) — boot disk deleted with VM           │
│     ○ Unchecked — boot disk persists after VM deletion       │
│                                                                   │
│   Encryption:                                                   │
│   ● Google-managed encryption key (default)                   │
│   ○ Customer-managed encryption key (CMEK via Cloud KMS)     │
│   ○ Customer-supplied encryption key (CSEK — you manage)     │
│                                                                   │
│ ── Identity and API access ──                                   │
│ Service account:                                                │
│   [sa-web-prod@project.iam.gserviceaccount.com ▼]             │
│   ⚡ CRITICAL: Service account = how VM accesses GCP APIs.     │
│   Like AWS IAM instance profile / EC2 role.                    │
│   ⚠️ DON'T use the default compute service account in prod!   │
│   Create a dedicated SA with least-privilege permissions.     │
│                                                                   │
│ Access scopes:                                                  │
│   ● Allow full access to all Cloud APIs                       │
│     (Access controlled by IAM roles on the service account)   │
│   ○ Set access for each API (legacy, use IAM instead)        │
│   ⚡ Always use "Allow full access" + restrict via IAM roles! │
│                                                                   │
│ ── Firewall ──                                                  │
│ ☑ Allow HTTP traffic (adds network tag "http-server")        │
│ ☑ Allow HTTPS traffic (adds network tag "https-server")      │
│ ⚡ These add VPC firewall rules matching these network tags.   │
│ ⚠️ For production: Don't use these — create specific rules!   │
│                                                                   │
│ ▸ Advanced options (expand)                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           ADVANCED OPTIONS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Networking ──                                                │
│                                                                   │
│ Network tags: [web-prod] [internal]                            │
│ ⚡ Used in firewall rules — target VMs by tag!                  │
│ e.g., "Allow 443 from 0.0.0.0/0 to tag:web-prod"             │
│                                                                   │
│ Hostname: [web-prod-01.internal]                               │
│                                                                   │
│ Network interfaces:                                             │
│ ┌───────────────────────────────────────────────────────────┐  │
│ │ Network: [vpc-prod ▼]                                     │  │
│ │ Subnetwork: [subnet-web-asia-south1 ▼]                   │  │
│ │ Primary internal IPv4: [Ephemeral ▼]                     │  │
│ │   ○ Ephemeral (auto-assigned, changes on delete)         │  │
│ │   ○ Reserve static internal IP                            │  │
│ │                                                           │  │
│ │ External IPv4 address:                                    │  │
│ │   ○ None (no public IP — recommended for production!)   │  │
│ │   ○ Ephemeral (temp public IP, changes on stop/start)   │  │
│ │   ○ Reserve static external IP                            │  │
│ │   ⚡ Production VMs: No external IP, access via IAP or LB│  │
│ │                                                           │  │
│ │ Network Service Tier:                                     │  │
│ │   ● Premium (Google's backbone — recommended)            │  │
│ │   ○ Standard (public internet — cheaper, higher latency)│  │
│ │                                                           │  │
│ │ IP forwarding: ☐ (enable for NAT/router VMs)            │  │
│ └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│ ── Management ──                                                │
│                                                                   │
│ Description: [Production web server for main application]     │
│                                                                   │
│ Deletion protection: ☑ Enable                                 │
│ ⚡ Prevents accidental VM deletion!                             │
│                                                                   │
│ Reservations:                                                   │
│   ● Automatically use created reservation                     │
│   ○ Select a specific reservation                             │
│   ○ Do not use                                                │
│                                                                   │
│ Availability policies:                                         │
│   On host maintenance:                                         │
│     ● Migrate (live migration — RECOMMENDED)                 │
│     ○ Terminate (stop during maintenance)                    │
│   Automatic restart: ● On (restart if crashed)               │
│   ⚡ Live migration: Google moves your VM to another host     │
│     during maintenance — zero downtime!                       │
│     (AWS and Azure don't offer this for all instance types)  │
│                                                                   │
│ Provisioning model:                                            │
│   ● Standard (on-demand — always available)                  │
│   ○ Spot (up to 91% cheaper — can be preempted!)            │
│                                                                   │
│ VM provisioning model comparison:                              │
│ ┌──────────────┬────────────┬──────────────────────────────┐  │
│ │ Type         │ Discount   │ Details                       │  │
│ ├──────────────┼────────────┼──────────────────────────────┤  │
│ │ Standard     │ 0%         │ Always available, pay/sec    │  │
│ │ Spot VM      │ 60-91%     │ Can be preempted anytime     │  │
│ │              │            │ No guaranteed runtime         │  │
│ │              │            │ 30-second warning             │  │
│ │ Preemptible  │ 60-91%     │ Legacy (use Spot instead)   │  │
│ │ (legacy)     │            │ Max 24 hours, then stopped   │  │
│ └──────────────┴────────────┴──────────────────────────────┘  │
│                                                                   │
│ ── Security ──                                                  │
│                                                                   │
│ Shielded VM:                                                   │
│   ☑ Turn on Secure Boot                                       │
│   ☑ Turn on vTPM                                              │
│   ☑ Turn on Integrity Monitoring                              │
│   ⚡ Enable all three for production!                          │
│   Prevents rootkits and boot-level malware.                   │
│                                                                   │
│ SSH Keys:                                                      │
│   ☐ Block project-wide SSH keys                               │
│   (If checked, only instance-level keys work)                 │
│                                                                   │
│   Add SSH key: [paste public key]                             │
│   Or: Use OS Login (RECOMMENDED — IAM-based SSH access)      │
│                                                                   │
│ ── Disks ──                                                    │
│                                                                   │
│ Additional disks:                                              │
│ [Add new disk]                                                 │
│   Name: [disk-data-web-prod-01]                               │
│   Type: [Balanced persistent disk ▼]                          │
│   Size: [100] GB                                               │
│   Mode: ● Read/write  ○ Read-only                            │
│   Deletion rule: ○ Delete  ● Keep (persist after VM delete)  │
│   Encryption: Google-managed                                   │
│   Snapshot schedule: [daily-snapshot ▼]                        │
│                                                                   │
│ [Attach existing disk] (re-attach a disk from another VM)    │
│                                                                   │
│ Local SSDs: [0 ▼] (0, 1, 2, 4, 8, 16, 24)                  │
│   ⚠️ Data LOST on stop/preemption! Ephemeral only!             │
│   375 GB per local SSD, up to 9 TB total                     │
│   ~1M IOPS — extremely fast                                   │
│                                                                   │
│ ── Automation ──                                                │
│                                                                   │
│ Startup script:                                                │
│ #!/bin/bash                                                    │
│ apt-get update && apt-get install -y nodejs nginx              │
│ systemctl enable nginx && systemctl start nginx                │
│ gsutil cp gs://config-bucket/web/app.conf /etc/nginx/         │
│ echo "Hello from $(hostname)" > /var/www/html/index.html      │
│                                                                   │
│ ⚡ Startup script runs on EVERY boot (not just first like AWS │
│    user data default). Use metadata to control run-once logic.│
│                                                                   │
│ Metadata:                                                      │
│ ┌──────────────────────┬──────────────────────────────────┐   │
│ │ Key                  │ Value                             │   │
│ ├──────────────────────┼──────────────────────────────────┤   │
│ │ startup-script       │ (inline script or URL)           │   │
│ │ startup-script-url   │ gs://bucket/startup.sh           │   │
│ │ shutdown-script      │ (cleanup on shutdown)            │   │
│ │ enable-oslogin       │ TRUE                              │   │
│ │ environment          │ prod                              │   │
│ └──────────────────────┴──────────────────────────────────┘   │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
│ ⚡ After creation:                                                │
│ ├── VM gets internal IP (e.g., 10.0.1.5)                     │
│ ├── External IP (if assigned): 35.200.xx.xx                  │
│ ├── SSH via console, gcloud, or IAP                          │
│ └── Mount additional disks, format, configure                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: SSH Access Methods

```
┌─────────────────────────────────────────────────────────────────────┐
│           SSH ACCESS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: Console SSH (browser-based)                               │
│ ├── VM instances → SSH button → opens SSH in browser              │
│ ├── No key pair management needed                                  │
│ ├── Google pushes temporary key automatically                     │
│ └── ⚡ Great for quick access                                       │
│                                                                       │
│ Method 2: gcloud SSH                                                │
│ gcloud compute ssh web-prod-01 --zone=asia-south1-a               │
│ ├── Auto-creates and manages SSH keys (~/.ssh/google_compute_*)  │
│ ├── Pushes public key to VM metadata                              │
│ ├── Works through IAP tunnel (no public IP needed!)              │
│ └── ⚡ Recommended for daily use                                   │
│                                                                       │
│ Method 3: IAP (Identity-Aware Proxy) SSH Tunnel                   │
│ gcloud compute ssh web-prod-01 \                                   │
│   --zone=asia-south1-a \                                           │
│   --tunnel-through-iap                                             │
│ ├── No public IP required!                                        │
│ ├── No firewall rule for SSH needed!                              │
│ ├── Traffic tunneled through Google's IAP service                │
│ ├── Access controlled by IAM (roles/iap.tunnelResourceAccessor) │
│ ├── All sessions logged in Cloud Audit Logs                      │
│ └── ⚡ MOST SECURE — recommended for production!                  │
│     (Like AWS SSM Session Manager)                                │
│                                                                       │
│ Method 4: OS Login (IAM-managed SSH)                                │
│ ├── Enable: Set metadata enable-oslogin=TRUE on project or VM   │
│ ├── SSH keys linked to Google/Cloud Identity user accounts       │
│ ├── IAM roles control SSH access:                                 │
│ │   ├── roles/compute.osLogin: SSH as non-root                   │
│ │   └── roles/compute.osAdminLogin: SSH as root (sudo)          │
│ ├── 2FA support (OS Login with 2FA)                               │
│ ├── No managing SSH keys in VM metadata                          │
│ └── ⚡ Best for organizations — centralized SSH access via IAM   │
│                                                                       │
│ ⚡ Best practice:                                                     │
│    Enable OS Login project-wide + use IAP tunnels.                │
│    No public IPs, no SSH firewall rules, full audit trail.       │
│    IAM controls who can SSH, audit logs track sessions.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Persistent Disks

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERSISTENT DISKS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Persistent Disks are network-attached block storage (like AWS EBS).│
│                                                                       │
│ ┌─────────────────┬──────────┬──────────┬──────────┬─────────────┐│
│ │ Type            │ IOPS (r) │ IOPS (w) │ Thruput  │ Cost/GB/mo  ││
│ ├─────────────────┼──────────┼──────────┼──────────┼─────────────┤│
│ │ pd-standard     │ 0.75/GB  │ 1.5/GB   │ Variable │ $0.04       ││
│ │ (HDD)           │          │          │          │ ⚡ Cheapest   ││
│ │                 │          │          │          │             ││
│ │ pd-balanced     │ 6/GB     │ 6/GB     │ Variable │ $0.10       ││
│ │ (SSD)           │ max 80K  │ max 30K  │          │ ⚡ Default    ││
│ │                 │          │          │          │             ││
│ │ pd-ssd          │ 30/GB    │ 30/GB    │ Variable │ $0.17       ││
│ │ (SSD)           │ max 100K │ max 100K │          │ Fast DB     ││
│ │                 │          │          │          │             ││
│ │ pd-extreme      │ custom   │ custom   │ custom   │ $0.125 +    ││
│ │ (SSD)           │ max 120K │ max 120K │          │ $0.006/IOPS ││
│ │                 │          │          │          │ SAP, Oracle ││
│ │                 │          │          │          │             ││
│ │ Hyperdisk       │ custom   │ custom   │ custom   │ Varies      ││
│ │ Balanced/Extreme│ max 350K │ max 350K │          │ Latest gen  ││
│ └─────────────────┴──────────┴──────────┴──────────┴─────────────┘│
│                                                                       │
│ Key features:                                                        │
│ ├── Zonal: Disk in one zone (same zone as VM)                    │
│ ├── Regional: Replicated across 2 zones (HA for critical data)   │
│ ├── Resize on the fly: Grow disk while VM is running             │
│ ├── Snapshot: Point-in-time backup (stored in Cloud Storage)    │
│ ├── Multi-reader: Multiple VMs can mount same disk read-only    │
│ ├── Performance scales with size (bigger disk = more IOPS)      │
│ └── Always encrypted at rest (Google-managed or CMEK)           │
│                                                                       │
│ ⚡ 99% of the time, use pd-balanced. Use pd-ssd for databases.    │
│                                                                       │
│ Local SSD (instance store):                                         │
│ ├── Physically attached NVMe SSD                                  │
│ ├── 375 GB per disk, up to 24 disks (9 TB total)                │
│ ├── ~1 million IOPS — extremely fast                             │
│ ├── Data LOST when VM stops, is preempted, or host fails        │
│ ├── Cannot be detached or re-attached                             │
│ ├── Available on specific machine types                          │
│ └── Use for: Scratch data, temp cache, high-throughput temp     │
│                                                                       │
│ Mount additional disk on Linux:                                     │
│ sudo mkfs.ext4 -m 0 -E lazy_itable_init=0 /dev/sdb              │
│ sudo mkdir -p /mnt/data                                            │
│ sudo mount /dev/sdb /mnt/data                                     │
│ echo '/dev/sdb /mnt/data ext4 defaults 0 2' | sudo tee -a       │
│   /etc/fstab                                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Snapshots

```
┌─────────────────────────────────────────────────────────────────────┐
│           SNAPSHOTS                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Point-in-time backup of a persistent disk.                   │
│ ├── Incremental (after first, only changed blocks)                │
│ ├── Stored in Cloud Storage (cross-regional by default)          │
│ ├── Create disk from snapshot in any zone/region                 │
│ └── Automate with snapshot schedules                              │
│                                                                       │
│ Create:                                                              │
│ Compute Engine → Snapshots → Create snapshot                       │
│   Name: [snap-web-prod-01-2026-05-16]                              │
│   Source disk: [web-prod-01 ▼]                                    │
│   Location:                                                         │
│     ● Multi-regional (US, EU, or Asia — highest durability)      │
│     ○ Regional (cheaper, single region)                           │
│   Labels: backup-type=manual, date=2026-05-16                     │
│                                                                       │
│ Snapshot schedule:                                                    │
│ Compute Engine → Snapshot schedules → Create                       │
│   Name: [schedule-daily-web]                                       │
│   Region: [asia-south1]                                             │
│   Frequency: ● Daily                                               │
│   Start time: [02:00] (2 AM — low traffic)                        │
│   Retention: [14] days (auto-delete after 14 days)                │
│   Deletion rule: ● Keep snapshots beyond retention (safer)       │
│   Storage location: Multi-regional (asia)                          │
│   Labels: managed-by=schedule                                       │
│                                                                       │
│ Attach schedule to disk:                                             │
│ Compute Engine → Disks → Select disk → Edit →                     │
│ Snapshot schedule: [schedule-daily-web ▼]                          │
│                                                                       │
│ ⚡ Always set up snapshot schedules for production data disks!     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Spot VMs

```
┌─────────────────────────────────────────────────────────────────────┐
│           SPOT VMs (formerly Preemptible)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Excess Compute Engine capacity at 60-91% discount.           │
│ Google can reclaim them at any time with 30-second warning.       │
│                                                                       │
│ ┌──────────────────┬──────────────────┬─────────────────────────┐ │
│ │ Feature          │ Spot VM          │ Preemptible (legacy)    │ │
│ ├──────────────────┼──────────────────┼─────────────────────────┤ │
│ │ Max runtime      │ No limit         │ 24 hours max            │ │
│ │ Preemption       │ Based on capacity│ After 24h + any time    │ │
│ │ Pricing          │ Dynamic pricing  │ Fixed discount          │ │
│ │ Availability     │ No guarantee     │ No guarantee            │ │
│ │ Warning          │ 30 seconds       │ 30 seconds              │ │
│ │ Recommended      │ ✅ Yes           │ ❌ Use Spot instead     │ │
│ └──────────────────┴──────────────────┴─────────────────────────┘ │
│                                                                       │
│ Good for:                                                            │
│ ├── Batch processing (video encoding, data pipelines)            │
│ ├── CI/CD build runners                                           │
│ ├── Big data / Dataproc (Hadoop/Spark)                           │
│ ├── Dev/test environments                                         │
│ ├── Fault-tolerant workloads with checkpointing                  │
│ └── Training ML models (with checkpointing)                      │
│                                                                       │
│ ⚠️ NEVER use for:                                                   │
│ ├── Production web servers (use MIG + standard VMs)              │
│ ├── Databases (data loss risk)                                    │
│ └── Stateful apps without replication                             │
│                                                                       │
│ Handle preemption:                                                   │
│ ├── Shutdown script: Save state, flush logs, checkpoint          │
│ │   metadata: shutdown-script = "gsutil cp /tmp/state gs://..."  │
│ ├── MIG auto-recreates Spot VMs after preemption                 │
│ └── Use multiple zones + instance types for Spot diversity       │
│                                                                       │
│ ⚡ Comparison:                                                       │
│ ├── GCP Spot VM ≈ AWS Spot Instance ≈ Azure Spot VM            │
│ ├── GCP: 30-second warning, no max runtime                      │
│ ├── AWS: 2-minute warning, no max runtime                       │
│ └── Azure: 30-second warning, eviction types vary                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Sole-Tenant Nodes

```
┌─────────────────────────────────────────────────────────────────────┐
│           SOLE-TENANT NODES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Dedicated physical server exclusively for your VMs.          │
│ No other customer's VMs run on the same hardware.                 │
│                                                                       │
│ Use cases:                                                           │
│ ├── License compliance (BYOL — Windows Server, Oracle, SQL)     │
│ ├── Regulatory/compliance (data isolation requirements)          │
│ ├── Performance isolation (no noisy neighbors)                   │
│ └── Security (physical separation)                                │
│                                                                       │
│ AWS equivalent: Dedicated Hosts                                    │
│ Azure equivalent: Dedicated Hosts                                  │
│                                                                       │
│ How it works:                                                        │
│ 1. Create a node template (defines machine type family)          │
│ 2. Create a node group (pool of sole-tenant servers)             │
│ 3. Create VMs targeting the node group                            │
│ 4. VMs run exclusively on your dedicated nodes                   │
│                                                                       │
│ ⚡ Most companies DON'T need sole-tenant nodes.                     │
│    Only use for BYOL licensing or strict compliance.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Purchasing Discounts

```
┌─────────────────────────────────────────────────────────────────────┐
│           PURCHASING OPTIONS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬──────────┬───────────┬───────────────────────┐│
│ │ Option           │ Discount │ Commitment│ Details               ││
│ ├──────────────────┼──────────┼───────────┼───────────────────────┤│
│ │ On-Demand        │ 0%       │ None      │ Pay per second,       ││
│ │                  │          │           │ min 1 minute          ││
│ │                  │          │           │                       ││
│ │ Sustained Use    │ Up to    │ None      │ AUTOMATIC! No action  ││
│ │ Discount (SUD)   │ 30%     │ (auto)    │ needed. Use VM >25%   ││
│ │                  │          │           │ of month → discount   ││
│ │                  │          │           │ increases gradually.  ││
│ │                  │          │           │ ⚡ Free money!          ││
│ │                  │          │           │                       ││
│ │ Committed Use    │ Up to    │ 1 or 3    │ Commit to $/hr for   ││
│ │ Discount (CUD)   │ 57%     │ years     │ vCPU + memory.        ││
│ │                  │          │           │ Applies to any VM     ││
│ │                  │          │           │ type (resource-based).││
│ │                  │          │           │ ⚡ Like AWS Savings Plan││
│ │                  │          │           │                       ││
│ │ Spot VM          │ 60-91%  │ None      │ Can be preempted.     ││
│ │                  │          │           │ For batch, CI/CD.     ││
│ └──────────────────┴──────────┴───────────┴───────────────────────┘│
│                                                                       │
│ Sustained Use Discount (SUD) — GCP exclusive:                      │
│ ├── 0-25% usage: Full price                                       │
│ ├── 25-50% usage: 20% discount on incremental                   │
│ ├── 50-75% usage: 40% discount on incremental                   │
│ ├── 75-100% usage: 60% discount on incremental                  │
│ ├── Net: ~30% discount for full month usage                      │
│ └── ⚡ Applies AUTOMATICALLY. No sign-up needed!                  │
│     (Only for N1, N2D. Not for E2, C2, etc.)                    │
│                                                                       │
│ Committed Use Discounts (CUD):                                      │
│ ├── Resource-based: Commit to X vCPUs + Y GB memory             │
│ ├── Spend-based: Commit to $/hr (for specific services)         │
│ ├── 1 year: Up to 37% discount                                   │
│ ├── 3 year: Up to 57% discount                                   │
│ ├── Applies across projects in same billing account              │
│ ├── Any machine family/size (flexible)                           │
│ └── ⚡ Recommended for steady-state production workloads          │
│                                                                       │
│ ⚡ Cost optimization strategy:                                       │
│ ├── Baseline (always-on): CUD (1yr resource-based)              │
│ ├── Variable demand: Standard VMs (get SUD automatically)       │
│ ├── Fault-tolerant: Spot VMs                                     │
│ └── Right-size: Use custom machine types                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform Example

```hcl
# VM Instance
resource "google_compute_instance" "web" {
  name         = "web-prod-01"
  machine_type = "n2-standard-4"
  zone         = "asia-south1-a"

  tags = ["web-prod", "https-server"]

  labels = {
    environment = "prod"
    team        = "backend"
    app         = "web-server"
  }

  boot_disk {
    initialize_params {
      image  = data.google_compute_image.web_prod.self_link
      size   = 20
      type   = "pd-balanced"
      labels = {
        environment = "prod"
      }
    }
  }

  attached_disk {
    source      = google_compute_disk.data.self_link
    device_name = "data-disk"
    mode        = "READ_WRITE"
  }

  network_interface {
    network    = google_compute_network.prod.id
    subnetwork = google_compute_subnetwork.web.id

    # No external IP for production — access via IAP
    # access_config {} ← uncomment for external IP
  }

  service_account {
    email  = google_service_account.web.email
    scopes = ["cloud-platform"]
  }

  metadata = {
    enable-oslogin = "TRUE"
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    apt-get update && apt-get install -y nodejs nginx
    gsutil cp gs://config-bucket/web/app.conf /etc/nginx/conf.d/
    systemctl enable nginx && systemctl start nginx
  EOF

  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  scheduling {
    automatic_restart   = true
    on_host_maintenance = "MIGRATE"
    preemptible         = false
    provisioning_model  = "STANDARD"
  }

  deletion_protection = true

  allow_stopping_for_update = true
}

# Data disk
resource "google_compute_disk" "data" {
  name  = "disk-data-web-prod-01"
  type  = "pd-balanced"
  zone  = "asia-south1-a"
  size  = 100

  labels = {
    environment = "prod"
  }
}

# Latest custom image from family
data "google_compute_image" "web_prod" {
  family  = "img-web-prod"
  project = var.project_id
}

# Service account
resource "google_service_account" "web" {
  account_id   = "sa-web-prod"
  display_name = "Web Server Service Account"
}

resource "google_project_iam_member" "web_logging" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.web.email}"
}

resource "google_project_iam_member" "web_monitoring" {
  project = var.project_id
  role    = "roles/monitoring.metricWriter"
  member  = "serviceAccount:${google_service_account.web.email}"
}

# Snapshot schedule
resource "google_compute_resource_policy" "daily_snapshot" {
  name   = "schedule-daily-web"
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
      storage_locations = ["asia"]
      labels = {
        managed-by = "schedule"
      }
    }
  }
}

resource "google_compute_disk_resource_policy_attachment" "data_snapshot" {
  name = google_compute_resource_policy.daily_snapshot.name
  disk = google_compute_disk.data.name
  zone = "asia-south1-a"
}

# Spot VM (for batch processing)
resource "google_compute_instance" "batch" {
  name         = "batch-worker-01"
  machine_type = "c3-standard-8"
  zone         = "asia-south1-a"
  tags         = ["batch"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 50
      type  = "pd-balanced"
    }
  }

  network_interface {
    network    = google_compute_network.prod.id
    subnetwork = google_compute_subnetwork.batch.id
  }

  service_account {
    email  = google_service_account.batch.email
    scopes = ["cloud-platform"]
  }

  scheduling {
    preemptible                 = true
    provisioning_model          = "SPOT"
    automatic_restart           = false
    on_host_maintenance         = "TERMINATE"
    instance_termination_action = "STOP"
  }

  metadata = {
    shutdown-script = <<-SCRIPT
      #!/bin/bash
      gsutil cp /tmp/checkpoint.dat gs://batch-bucket/checkpoints/
    SCRIPT
  }
}

# Static external IP (if needed)
resource "google_compute_address" "web" {
  name         = "ip-web-prod"
  region       = "asia-south1"
  network_tier = "PREMIUM"
}
```

---

## Part 10: gcloud CLI Reference

```bash
# Create VM
gcloud compute instances create web-prod-01 \
  --zone=asia-south1-a \
  --machine-type=n2-standard-4 \
  --image-family=img-web-prod \
  --image-project=my-project-id \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-balanced \
  --subnet=subnet-web-asia-south1 \
  --no-address \
  --tags=web-prod,https-server \
  --service-account=sa-web-prod@project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --metadata=enable-oslogin=TRUE \
  --metadata-from-file=startup-script=startup.sh \
  --shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring \
  --deletion-protection \
  --labels=environment=prod,team=backend

# Create Spot VM
gcloud compute instances create batch-worker-01 \
  --zone=asia-south1-a \
  --machine-type=c3-standard-8 \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB

# SSH into VM
gcloud compute ssh web-prod-01 --zone=asia-south1-a

# SSH via IAP tunnel (no public IP needed)
gcloud compute ssh web-prod-01 \
  --zone=asia-south1-a \
  --tunnel-through-iap

# Stop/start/delete
gcloud compute instances stop web-prod-01 --zone=asia-south1-a
gcloud compute instances start web-prod-01 --zone=asia-south1-a
gcloud compute instances delete web-prod-01 --zone=asia-south1-a

# List instances
gcloud compute instances list \
  --filter="labels.environment=prod" \
  --format="table(name,zone,machineType.basename(),status,networkInterfaces[0].networkIP)"

# Create custom machine type
gcloud compute instances create api-prod-01 \
  --zone=asia-south1-a \
  --custom-cpu=6 \
  --custom-memory=24GB

# Create disk
gcloud compute disks create disk-data-web-prod \
  --zone=asia-south1-a \
  --size=100GB \
  --type=pd-balanced

# Attach disk
gcloud compute instances attach-disk web-prod-01 \
  --zone=asia-south1-a \
  --disk=disk-data-web-prod \
  --device-name=data-disk

# Create snapshot
gcloud compute disks snapshot web-prod-01 \
  --zone=asia-south1-a \
  --snapshot-names=snap-web-prod-2026-05-16

# Create image from disk
gcloud compute images create img-web-prod-v1-2-3 \
  --source-disk=web-prod-01 \
  --source-disk-zone=asia-south1-a \
  --family=img-web-prod \
  --labels=version=1-2-3

# Resize disk (grow only)
gcloud compute disks resize disk-data-web-prod \
  --zone=asia-south1-a \
  --size=200GB
```

---

## Part 11: Real-World Patterns

### Startup

```
VMs: 2 e2-standard-2 or c4a-standard-2 (Arm) 
Image: Debian 12 or Ubuntu 24.04 LTS
Access: gcloud SSH via IAP (no public IPs, no SSH keys to manage)
Service account: Dedicated SA with logging + monitoring roles
Boot disk: 20 GB pd-balanced
Data disk: None (use Cloud SQL or Firestore)
Startup script: Install app, pull config from Secret Manager
Network: No external IP, behind HTTP(S) LB
Spot VMs: For CI/CD runners only

Monitoring:
├── Ops Agent (CPU, memory, disk metrics)
├── Cloud Logging (app logs)
└── Uptime checks (HTTP health)

Cost: ~$50/month (2x e2-standard-2 with SUD)
```

### Mid-Size

```
Web tier: 4 n2-standard-4 (MIG with autoscaler)
API tier: 4 n2-standard-4 (MIG with autoscaler)
Worker tier: 2 c3-standard-4 (Spot VMs via MIG)

Image: Custom golden image (Packer-built, weekly)
Image family: img-web-prod, img-api-prod
Access: OS Login + IAP tunnels (no SSH keys, no public IPs)
Service accounts: Per-tier (sa-web, sa-api, sa-worker)
Boot disk: 20 GB pd-balanced
Data disk: 100 GB pd-balanced (for logs, temp data)
Snapshots: Daily, 14-day retention

Purchasing:
├── CUD: 1-year resource-based (covers 60% baseline)
├── SUD: Automatic for N2 VMs running >25% month
├── Spot: Worker tier + CI/CD runners
└── Custom machine types for right-sizing

Monitoring:
├── Ops Agent on all VMs
├── Custom dashboards per tier
├── Alert policies (CPU, memory, disk, health check)
└── VM Manager for patching

Cost: ~$300-600/month (CUD + SUD + Spot savings)
```

### Enterprise

```
Web/API: n2-standard-8 in MIGs (3 zones, 3 regions)
Database: m3-ultramem (SAP HANA) or use Cloud SQL
ML: a2-highgpu-1g (GPU) — Spot for training, standard for inference
Batch: c3-highcpu (Spot VMs, MIG with autoscaler)

Images:
├── Hardened base image (CIS benchmark, weekly rebuild)
├── Per-service golden images (app pre-baked)
├── Image family per service (auto-latest)
└── Shared in Org (Shared VPC host project)

Storage:
├── Boot: 30 GB pd-balanced
├── Data: 500 GB pd-ssd (databases)
├── Snapshots: Daily (14d) + weekly (90d) + cross-region copy
└── Regional PD for critical stateful workloads

Security:
├── OS Login with 2FA (project-wide)
├── Shielded VMs (all instances)
├── IAP tunnels only (no public IPs, no SSH port)
├── Confidential VMs (for regulated data)
├── CMEK encryption (Cloud KMS managed keys)
├── VM Manager: Patch management, compliance
├── OS vulnerability scanning (Security Command Center)
└── Organization policies: Block external IPs, require Shielded VM

Purchasing:
├── CUD: 3-year resource-based (covers 50-60% baseline)
├── SUD: Automatic for N1/N2D
├── Spot: 40% of batch/ML training fleet
├── Sole-tenant nodes: Windows Server BYOL
└── Savings: 45-60% vs all on-demand

Cost: ~$5,000-50,000/month
```

---

## Part 12: Console Walkthrough: Managing & Deleting VMs

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING VMs FROM THE CONSOLE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Compute Engine → VM instances                            │
│                                                                       │
│ ── Stop / Start / Suspend / Resume ──                              │
│                                                                       │
│ 1. Select VM (checkbox) or click VM name → top toolbar:           │
│    ┌──────┬───────┬─────────┬────────┬──────────┬────────┐       │
│    │ Start│ Stop  │ Suspend │ Resume │ Reset    │ Delete │       │
│    └──────┴───────┴─────────┴────────┴──────────┴────────┘       │
│                                                                       │
│ Or: Click ⋮ (three dots) on the VM row → choose action           │
│                                                                       │
│ States explained:                                                    │
│ ├── RUNNING → VM is active, you're being charged                 │
│ ├── STOPPED → VM is off, NO compute charges                      │
│ │   (still charged for disks + static IPs)                        │
│ ├── SUSPENDED → VM state saved to disk, NO compute charges       │
│ │   (faster resume than stopped, charged for suspended RAM)      │
│ └── TERMINATED → Spot VM was preempted or VM crashed             │
│                                                                       │
│ ⚡ Stopped vs Suspended:                                             │
│    Stopped = cold start (boots from scratch, 30-60 seconds)      │
│    Suspended = warm resume (restores RAM state, ~10-15 seconds)  │
│    Use suspend for dev VMs you pause during lunch/night!         │
│                                                                       │
│ ── Change Machine Type (Resize VM) ──                              │
│                                                                       │
│ ⚠️ You CANNOT change machine type while VM is running!              │
│                                                                       │
│ Steps:                                                               │
│ 1. Stop the VM (VM instances → select VM → Stop → confirm)      │
│ 2. Wait for status to show TERMINATED/STOPPED                    │
│ 3. Click VM name → Edit (pencil icon at top)                     │
│ 4. Machine configuration section:                                  │
│    Series: [N2 ▼]                                                 │
│    Machine type: [n2-standard-8 ▼] (was n2-standard-4)          │
│    Or switch to Custom: Cores [8] Memory [32] GB                 │
│ 5. Click Save                                                      │
│ 6. Start the VM (toolbar → Start)                                 │
│                                                                       │
│ ⚡ Total downtime: 2-5 minutes (stop + change + start)             │
│    Schedule resize during maintenance windows!                    │
│                                                                       │
│ ── Add / Remove Network Tags ──                                    │
│                                                                       │
│ Network tags control which firewall rules apply to a VM.          │
│                                                                       │
│ Steps:                                                               │
│ 1. Click VM name → Edit                                           │
│ 2. Scroll to "Networking" section                                 │
│ 3. Network tags: [web-prod] [internal] [+ Add tag]              │
│    Type new tag and press Enter                                   │
│    Click ✕ on a tag to remove it                                  │
│ 4. Click Save                                                      │
│                                                                       │
│ ⚡ Tags can be changed while VM is RUNNING — no restart needed!    │
│ ⚠️ Adding/removing a tag instantly changes firewall behavior!       │
│    Double-check your firewall rules before changing tags.         │
│                                                                       │
│ ── Delete a VM ──                                                   │
│                                                                       │
│ Steps:                                                               │
│ 1. VM instances → select VM (checkbox)                            │
│ 2. Click Delete (trash icon) in toolbar                           │
│    Or: ⋮ (three dots) → Delete                                   │
│ 3. Confirmation dialog: Type VM name or click Delete             │
│                                                                       │
│ What happens to disks on deletion:                                  │
│ ┌────────────────────┬─────────────────────────────────────────┐  │
│ │ Disk Type          │ Behavior                                │  │
│ ├────────────────────┼─────────────────────────────────────────┤  │
│ │ Boot disk          │ Deleted (default) — unless "Delete      │  │
│ │                    │ boot disk when instance is deleted"     │  │
│ │                    │ was unchecked during creation           │  │
│ │ Additional disks   │ Depends on deletion rule set at        │  │
│ │                    │ attach time (Delete or Keep)            │  │
│ │ Local SSDs         │ ALWAYS deleted (ephemeral storage)     │  │
│ └────────────────────┴─────────────────────────────────────────┘  │
│                                                                       │
│ ⚠️ Deletion protection: If enabled, you must first:                 │
│    VM details → Edit → uncheck "Enable deletion protection"      │
│    → Save → then delete                                           │
│                                                                       │
│ ⚡ Pro tip: Before deleting, create a machine image for backup!     │
│    Compute Engine → Machine images → Create → Source VM          │
│                                                                       │
│ ── View Serial Console Output ──                                    │
│                                                                       │
│ The serial console shows boot messages, kernel output, and        │
│ errors — even when SSH is not working.                             │
│                                                                       │
│ Steps:                                                               │
│ 1. Click VM name → Details tab                                    │
│ 2. Scroll down → "Serial port 1 (console)" section               │
│    Or: Logs section → "Serial port 1 output"                     │
│ 3. View boot logs, kernel messages, startup script output        │
│                                                                       │
│ Interactive serial console:                                         │
│ 1. VM must have metadata: serial-port-enable=TRUE                │
│ 2. VM details → "Connect to serial console"                      │
│ 3. Opens interactive terminal (even if SSH is broken!)           │
│                                                                       │
│ ⚡ Serial console is your best friend when:                         │
│    ├── VM won't boot (kernel panic, disk errors)                 │
│    ├── SSH is broken (bad config, firewall issues)               │
│    ├── Startup script fails (see error output)                   │
│    └── Network misconfiguration (can't reach VM)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Additional gcloud Commands

```bash
# ── Update VM metadata and tags ──

# Add/update metadata on a running VM
gcloud compute instances add-metadata web-prod-01 \
  --zone=asia-south1-a \
  --metadata=environment=staging,version=2.1.0

# Remove metadata keys
gcloud compute instances remove-metadata web-prod-01 \
  --zone=asia-south1-a \
  --keys=version

# Update tags (replaces ALL tags — include existing ones you want to keep)
gcloud compute instances add-tags web-prod-01 \
  --zone=asia-south1-a \
  --tags=web-prod,https-server,monitoring

# Remove specific tags
gcloud compute instances remove-tags web-prod-01 \
  --zone=asia-south1-a \
  --tags=monitoring

# ── Resize VM (change machine type) ──
# Step 1: Stop the VM
gcloud compute instances stop web-prod-01 --zone=asia-south1-a

# Step 2: Change machine type
gcloud compute instances set-machine-type web-prod-01 \
  --zone=asia-south1-a \
  --machine-type=n2-standard-8

# Step 3: Start the VM
gcloud compute instances start web-prod-01 --zone=asia-south1-a

# Change to custom machine type
gcloud compute instances set-machine-type web-prod-01 \
  --zone=asia-south1-a \
  --custom-cpu=6 \
  --custom-memory=24GB

# ── Move VM between zones ──
# ⚠️ Only works for VMs with no local SSDs and no GPUs
gcloud compute instances move web-prod-01 \
  --zone=asia-south1-a \
  --destination-zone=asia-south1-b

# ⚡ For moving across regions, use machine image instead:
#    1. Create machine image from VM
#    2. Create new VM from machine image in target region
#    3. Delete old VM

# ── Suspend and Resume ──
# Suspend (saves RAM to disk — faster resume than stop/start)
gcloud compute instances suspend web-prod-01 --zone=asia-south1-a

# Resume suspended VM
gcloud compute instances resume web-prod-01 --zone=asia-south1-a

# ⚡ Suspend is great for dev VMs you don't need overnight.
#    No compute charges while suspended (small disk charge for RAM state).

# ── Update instance properties ──
# Enable/disable deletion protection
gcloud compute instances update web-prod-01 \
  --zone=asia-south1-a \
  --deletion-protection

gcloud compute instances update web-prod-01 \
  --zone=asia-south1-a \
  --no-deletion-protection

# Update labels
gcloud compute instances update web-prod-01 \
  --zone=asia-south1-a \
  --update-labels=environment=staging,version=2.0

# Remove labels
gcloud compute instances update web-prod-01 \
  --zone=asia-south1-a \
  --remove-labels=version

# Update min-cpu-platform
gcloud compute instances update web-prod-01 \
  --zone=asia-south1-a \
  --min-cpu-platform="Intel Ice Lake"

# ── Snapshot commands ──
# List all snapshots
gcloud compute snapshots list \
  --format="table(name,diskSizeGb,status,sourceDisk.basename(),creationTimestamp)"

# List snapshots with filter
gcloud compute snapshots list \
  --filter="sourceDisk~web-prod"

# Describe a snapshot (full details)
gcloud compute snapshots describe snap-web-prod-2026-05-16

# Delete a snapshot
gcloud compute snapshots delete snap-web-prod-2026-05-16

# Create disk from snapshot (restore to any zone)
gcloud compute disks create disk-restored-web \
  --zone=asia-south1-b \
  --source-snapshot=snap-web-prod-2026-05-16 \
  --type=pd-balanced
```

---

## Part 14: Ops Agent Installation

```
┌─────────────────────────────────────────────────────────────────────┐
│           OPS AGENT (MONITORING + LOGGING AGENT)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The Ops Agent collects system metrics (CPU, memory, disk, network)│
│ and application logs from your VM and sends them to Cloud          │
│ Monitoring and Cloud Logging.                                      │
│                                                                       │
│ Without the Ops Agent, Cloud Monitoring only shows basic metrics  │
│ (CPU utilization, network traffic). With it, you get:              │
│ ├── Memory utilization (not available by default!)                │
│ ├── Disk I/O and usage                                             │
│ ├── Process-level metrics                                         │
│ ├── Application logs (nginx, apache, mysql, etc.)                │
│ └── Custom metrics and logs                                       │
│                                                                       │
│ ⚡ Think of it like AWS CloudWatch Agent — you need to install it  │
│    to get full visibility into your VMs.                           │
│                                                                       │
│ ── Install on a Single VM ──                                        │
│                                                                       │
│ SSH into the VM, then run:                                          │
│                                                                       │
│ curl -sSO https://dl.google.com/cloudagents/                      │
│   add-google-cloud-ops-agent-repo.sh                               │
│ sudo bash add-google-cloud-ops-agent-repo.sh --also-install       │
│                                                                       │
│ Verify installation:                                                 │
│ sudo systemctl status google-cloud-ops-agent"*"                   │
│                                                                       │
│ ── Install via gcloud (Fleet — Multiple VMs) ──                    │
│                                                                       │
│ # Install on all VMs matching a filter                              │
│ gcloud compute instances ops-agents policies create \              │
│   ops-agent-policy \                                                │
│   --agent-rules="type=ops-agent,version=current-major" \          │
│   --os-types=short-name=debian,version=12 \                       │
│   --instances-filter="labels.environment=prod"                     │
│                                                                       │
│ ── Install via Startup Script ──                                    │
│                                                                       │
│ Add to your VM startup script or Terraform metadata:               │
│ #!/bin/bash                                                        │
│ curl -sSO https://dl.google.com/cloudagents/                      │
│   add-google-cloud-ops-agent-repo.sh                               │
│ sudo bash add-google-cloud-ops-agent-repo.sh --also-install       │
│                                                                       │
│ ⚡ Best approach: Bake into your golden image so all new VMs      │
│    automatically have the agent pre-installed!                     │
│                                                                       │
│ ── Prerequisites ──                                                  │
│                                                                       │
│ The VM's service account needs these roles:                        │
│ ├── roles/monitoring.metricWriter (send metrics)                  │
│ ├── roles/logging.logWriter (send logs)                            │
│ └── roles/monitoring.viewer (optional — for dashboard access)    │
│                                                                       │
│ ⚡ Already set up if you used our Terraform example in Part 9!     │
│                                                                       │
│ ── Verify in Console ──                                              │
│                                                                       │
│ After installation:                                                  │
│ 1. Console → Monitoring → Dashboards → VM Instances               │
│ 2. You should see memory, disk, and process metrics               │
│ 3. Console → Logging → Logs Explorer                              │
│    Filter: resource.type="gce_instance"                           │
│    You should see system and application logs                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 15: Troubleshooting Common Issues

```
┌─────────────────────────────────────────────────────────────────────┐
│           TROUBLESHOOTING COMMON ISSUES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Problem 1: VM Not Starting ──                                     │
│                                                                       │
│ Symptom: VM stuck in STAGING or error on start.                   │
│                                                                       │
│ Possible causes:                                                     │
│                                                                       │
│ 1. Quota exceeded                                                   │
│    Error: "Quota 'CPUS' exceeded. Limit: 24.0"                   │
│    Fix:                                                              │
│    ├── Check quota: IAM & Admin → Quotas                         │
│    ├── Filter by: Compute Engine API → CPUs (or GPUs, etc.)     │
│    ├── Request increase: Click quota → Edit Quotas               │
│    └── Or try a different zone (quotas are per-region)           │
│    gcloud: gcloud compute regions describe asia-south1 \         │
│              --format="table(quotas.metric,quotas.limit,          │
│              quotas.usage)"                                       │
│                                                                       │
│ 2. Insufficient permissions                                        │
│    Error: "Required 'compute.instances.create' permission"        │
│    Fix:                                                              │
│    ├── User needs: roles/compute.instanceAdmin.v1 (or higher)   │
│    ├── Check: gcloud projects get-iam-policy PROJECT_ID          │
│    └── Grant: gcloud projects add-iam-policy-binding ...         │
│                                                                       │
│ 3. Resource not found (image, subnet, disk)                       │
│    Error: "The resource 'projects/.../images/...' was not found" │
│    Fix:                                                              │
│    ├── Check image exists: gcloud compute images list            │
│    ├── Check correct project: --image-project=correct-project   │
│    └── Check region/zone: subnet must be in same region as VM   │
│                                                                       │
│ 4. IP address exhaustion                                            │
│    Error: "IP space of subnet is exhausted"                       │
│    Fix:                                                              │
│    ├── Expand subnet range: gcloud compute networks subnets      │
│    │   expand-ip-range SUBNET --region=REGION                    │
│    │   --prefix-length=NEW_PREFIX                                 │
│    └── Or create VM in different subnet/zone                     │
│                                                                       │
│ ── Problem 2: Cannot SSH to VM ──                                    │
│                                                                       │
│ Symptom: SSH connection times out or is refused.                   │
│                                                                       │
│ Diagnosis flowchart:                                                 │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Can't SSH to VM                                             │   │
│ │   │                                                         │   │
│ │   ├── Does VM have external IP?                            │   │
│ │   │   ├── No → Use IAP tunnel:                            │   │
│ │   │   │   gcloud compute ssh VM --tunnel-through-iap      │   │
│ │   │   └── Ensure IAP firewall rule exists (35.235.240.0/20)│   │
│ │   │                                                         │   │
│ │   ├── Is there a firewall rule allowing SSH?               │   │
│ │   │   ├── Check: gcloud compute firewall-rules list       │   │
│ │   │   │   --filter="allowed[].ports:22"                    │   │
│ │   │   └── Create if missing:                               │   │
│ │   │       gcloud compute firewall-rules create allow-ssh   │   │
│ │   │         --allow=tcp:22 --source-ranges=YOUR_IP/32     │   │
│ │   │         --target-tags=TAG                              │   │
│ │   │                                                         │   │
│ │   ├── Is the VM running?                                   │   │
│ │   │   └── Check: gcloud compute instances list             │   │
│ │   │                                                         │   │
│ │   ├── Is SSH service running inside VM?                    │   │
│ │   │   └── Check via serial console (see Part 12)          │   │
│ │   │                                                         │   │
│ │   └── Are SSH keys correct?                                │   │
│ │       ├── OS Login: Check IAM roles                        │   │
│ │       │   (roles/compute.osLogin or osAdminLogin)          │   │
│ │       └── Metadata keys: Check project + instance metadata│   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ IAP tunnel troubleshooting:                                         │
│ ├── User needs: roles/iap.tunnelResourceAccessor                  │
│ ├── Firewall rule needed: Allow ingress from 35.235.240.0/20     │
│ │   (Google's IAP IP range) on TCP port 22                        │
│ ├── gcloud compute firewall-rules create allow-iap-ssh \         │
│ │     --allow=tcp:22 \                                            │
│ │     --source-ranges=35.235.240.0/20 \                          │
│ │     --network=vpc-prod                                          │
│ └── No external IP needed on the VM!                              │
│                                                                       │
│ ── Problem 3: Disk Full / Performance Issues ──                     │
│                                                                       │
│ Symptom: VM slow, apps crashing, "No space left on device".       │
│                                                                       │
│ Diagnosis:                                                           │
│ # Check disk usage (SSH into VM)                                   │
│ df -h                    # shows disk usage per mount              │
│ du -sh /var/log/*        # find large directories                 │
│ sudo journalctl --disk-usage  # systemd journal size              │
│                                                                       │
│ Fix — Grow disk (no downtime!):                                    │
│ # Step 1: Resize disk in GCP (can do while VM is running)        │
│ gcloud compute disks resize DISK_NAME \                           │
│   --zone=ZONE --size=100GB                                        │
│                                                                       │
│ # Step 2: Resize filesystem inside VM                              │
│ sudo resize2fs /dev/sda1       # ext4                              │
│ sudo xfs_growfs /               # xfs                              │
│                                                                       │
│ Performance issues:                                                  │
│ ├── Disk IOPS scales with size — bigger disk = more IOPS         │
│ │   (A 10 GB pd-balanced = 60 IOPS, 100 GB = 600 IOPS)          │
│ │   Fix: Increase disk size or upgrade to pd-ssd                 │
│ ├── Check Monitoring → VM instance → Disk metrics                │
│ │   Look for: high disk latency, IOPS at limit                   │
│ └── Local SSD: If you need extreme IOPS (~1M), use local SSD    │
│     ⚠️ Data is ephemeral! Only for scratch/cache.                  │
│                                                                       │
│ ── Problem 4: Serial Console Debugging ──                            │
│                                                                       │
│ When you can't SSH, the serial console is your lifeline.          │
│                                                                       │
│ View serial output (read-only, always available):                  │
│ gcloud compute instances get-serial-port-output VM_NAME \         │
│   --zone=ZONE                                                      │
│                                                                       │
│ Interactive serial console (requires setup):                        │
│ # Enable serial port access (project-wide or per-VM)              │
│ gcloud compute instances add-metadata VM_NAME \                   │
│   --zone=ZONE \                                                    │
│   --metadata=serial-port-enable=TRUE                               │
│                                                                       │
│ # Connect to interactive serial console                            │
│ gcloud compute connect-to-serial-port VM_NAME \                   │
│   --zone=ZONE                                                      │
│                                                                       │
│ Common things to look for in serial output:                        │
│ ├── Kernel panic → corrupted image, re-create from snapshot      │
│ ├── "No space left on device" → disk full, grow disk             │
│ ├── fsck errors → filesystem corruption, repair or restore       │
│ ├── Startup script errors → fix script, check Cloud Logging      │
│ ├── GRUB errors → boot disk issue, attach to rescue VM          │
│ └── Network config errors → check metadata, VPC settings        │
│                                                                       │
│ ⚡ Rescue VM technique:                                              │
│ 1. Stop the broken VM                                              │
│ 2. Detach its boot disk                                            │
│ 3. Create a new "rescue" VM                                       │
│ 4. Attach the broken disk as secondary disk to rescue VM         │
│ 5. SSH into rescue VM, mount the disk, fix files                 │
│ 6. Detach disk, re-attach to original VM as boot disk            │
│ 7. Start original VM                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| Machine families | E2 (cheap), N2/N4 (balanced), C3 (compute), M3 (memory), A2/G2 (GPU) |
| Arm instances | C4A, T2A — best price/perf (like AWS Graviton) |
| Custom types | Pick exact vCPU + memory — GCP unique feature |
| Images | Public (Debian, Ubuntu), custom, machine images, image families |
| Disk types | pd-standard (HDD), pd-balanced (SSD default), pd-ssd, pd-extreme |
| Local SSD | 375 GB NVMe, ephemeral, ~1M IOPS |
| Snapshots | Incremental, scheduled, multi-regional storage |
| SSH access | Console, gcloud, IAP tunnel (most secure), OS Login |
| Spot VMs | 60-91% discount, can be preempted, 30-second warning |
| SUD | Automatic up to 30% discount for sustained use |
| CUD | 1-3 year commitment, up to 57% discount |
| Shielded VM | Secure boot + vTPM + integrity monitoring |
| Live migration | Zero-downtime host maintenance — GCP exclusive |
| AWS equivalent | EC2 |
| Azure equivalent | Virtual Machines |

---

## What's Next?

In the next chapter, we'll cover Instance Groups & Autoscaling — automatically managing fleets of VMs.

→ Next: [Chapter 15: Instance Groups & Autoscaling](15-instance-groups-autoscaling.md)

---

*Last Updated: May 2026*
