# Chapter 15: Azure Virtual Machines

---

## Table of Contents

- [Overview](#overview)
- [Part 1: VM Series & Naming Convention](#part-1-vm-series--naming-convention)
- [Part 2: Creating a VM (Full Portal Walkthrough)](#part-2-creating-a-vm-full-portal-walkthrough)
- [Part 3: Availability Options](#part-3-availability-options)
- [Part 4: Spot VMs](#part-4-spot-vms)
- [Part 5: Custom Images & Shared Image Gallery](#part-5-custom-images--shared-image-gallery)
- [Part 6: VM Extensions](#part-6-vm-extensions)
- [Part 7: Terraform & Bicep Examples](#part-7-terraform--bicep-examples)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Virtual Machines (VMs) are on-demand, scalable compute resources. They give you full control over the operating system, installed software, and configuration — like having a physical server in the cloud. VMs are the most fundamental compute service in Azure.

```
What you'll learn:
├── VM Fundamentals
│   ├── VM types & series (General, Compute, Memory, Storage, GPU)
│   ├── Naming convention (D4s_v5, E8as_v5, etc.)
│   └── When to use VMs vs other compute (App Service, Functions, AKS)
├── Creating a VM (Full Portal Walkthrough)
│   ├── Basics (name, region, image, size, admin account)
│   ├── Disks (OS disk, data disks, disk types)
│   ├── Networking (VNet, subnet, NSG, public IP)
│   ├── Management (monitoring, auto-shutdown, backup)
│   ├── Advanced (extensions, custom data, proximity groups)
│   └── Tags
├── VM Sizes & Series (Decision Guide)
├── Images (Marketplace, custom, shared gallery)
├── Disks (Managed disks, types, performance tiers)
├── Availability (Sets, Zones, VMSS)
├── Spot VMs (up to 90% discount)
├── Custom Images & Shared Image Gallery
├── Extensions & Custom Script
├── az CLI, Terraform, Bicep examples
└── Real-world patterns
```

---

## Part 1: VM Series & Naming Convention

```
┌─────────────────────────────────────────────────────────────────────┐
│           VM SERIES (FAMILIES)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────┬────────────────┬─────────────────────────────────────┐  │
│ │ Series │ Optimized for  │ Use cases                           │  │
│ ├────────┼────────────────┼─────────────────────────────────────┤  │
│ │ B      │ Burstable      │ Dev/test, low-traffic web, CI/CD   │  │
│ │ D      │ General purpose│ Web servers, APIs, app servers      │  │
│ │ E      │ Memory         │ Databases, in-memory caching       │  │
│ │ F      │ Compute        │ Batch processing, gaming, ML infer.│  │
│ │ L      │ Storage        │ Big data, large databases, DW      │  │
│ │ M      │ Memory (large) │ SAP HANA, very large databases     │  │
│ │ N      │ GPU            │ ML training, rendering, HPC        │  │
│ │ H      │ HPC            │ Fluid dynamics, weather modeling   │  │
│ └────────┴────────────────┴─────────────────────────────────────┘  │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ NAMING CONVENTION:                                                   │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ Example: Standard_D4as_v5                                           │
│          ─────── ─┬─┬┬─ ──                                         │
│             │     │ ││ │  │                                          │
│             │     │ ││ │  └── Version (v5 = latest generation)    │
│             │     │ ││ └──── Sub-family:                          │
│             │     │ ││       s = Premium Storage capable          │
│             │     │ ││       d = local temp disk                   │
│             │     │ ││       nothing = no local temp disk         │
│             │     │ │└────── Additional capability:               │
│             │     │ │        a = AMD processor                    │
│             │     │ │        nothing = Intel processor            │
│             │     │ │        p = ARM processor                    │
│             │     │ └─────── vCPUs: 4                             │
│             │     └───────── Family: D (general purpose)         │
│             └─────────────── Size category                        │
│                                                                       │
│ Common sub-family letters:                                          │
│ ├── s: Supports Premium SSD (most prod VMs need this!)          │
│ ├── d: Has local temp NVMe disk (fast ephemeral storage)        │
│ ├── a: AMD processor (5-10% cheaper than Intel)                 │
│ ├── p: ARM (Ampere) processor (up to 20% cheaper)              │
│ ├── l: Low memory (less RAM per vCPU)                           │
│ ├── i: Isolated (dedicated physical server)                      │
│ └── t: Trusted launch (secure boot mandatory)                    │
│                                                                       │
│ Popular sizes for web/app workloads:                                │
│ ┌─────────────────┬──────┬────────┬───────────────────────────┐  │
│ │ Size            │ vCPU │ RAM    │ Monthly cost (India)      │  │
│ ├─────────────────┼──────┼────────┼───────────────────────────┤  │
│ │ B1s             │ 1    │ 1 GB   │ ~$8 (dev/test)           │  │
│ │ B2s             │ 2    │ 4 GB   │ ~$35 (small workloads)   │  │
│ │ D2s_v5          │ 2    │ 8 GB   │ ~$70 (general prod)      │  │
│ │ D4s_v5          │ 4    │ 16 GB  │ ~$140 (standard prod)    │  │
│ │ D4as_v5 (AMD)   │ 4    │ 16 GB  │ ~$125 (cheaper!)         │  │
│ │ D8s_v5          │ 8    │ 32 GB  │ ~$280 (medium prod)      │  │
│ │ E4s_v5          │ 4    │ 32 GB  │ ~$185 (memory-heavy)     │  │
│ │ F4s_v2          │ 4    │ 8 GB   │ ~$130 (compute-heavy)    │  │
│ └─────────────────┴──────┴────────┴───────────────────────────┘  │
│                                                                       │
│ Decision guide:                                                      │
│ ├── Dev/test, low traffic → B series (burstable, cheapest)      │
│ ├── Web servers, APIs → D series (balanced CPU:RAM)             │
│ ├── Databases, caching → E series (high RAM per vCPU)          │
│ ├── Batch, ML inference → F series (high CPU, lower RAM)       │
│ ├── Want cheaper → use AMD (a) or ARM (p) variants             │
│ └── Always use v5 (latest generation = best price/performance) │
│                                                                       │
│ ⚡ AWS equivalent: D series ≈ M-type, E series ≈ R-type,          │
│    F series ≈ C-type, B series ≈ T-type                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a VM (Full Portal Walkthrough)

```
Console → Virtual machines → Create → Azure virtual machine

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Project details ──                                               │
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-web-prod ▼]  [Create new]                     │
│                                                                       │
│ ── Instance details ──                                              │
│                                                                       │
│ Virtual machine name: [vm-web-prod-01]                             │
│ ⚡ Naming: vm-{purpose}-{env}-{number}                              │
│                                                                       │
│ Region: [Central India ▼]                                          │
│ ⚡ Choose closest to users. Affects available VM sizes & pricing.  │
│                                                                       │
│ Availability options:                                               │
│   ○ No infrastructure redundancy required                         │
│   ○ Availability zone (1, 2, or 3) ← RECOMMENDED for prod      │
│   ○ Availability set (legacy approach)                           │
│   ○ Virtual machine scale set                                    │
│                                                                       │
│ ⚡ Availability zones:                                               │
│   Zone 1, 2, or 3 — physically separate datacenters in region   │
│   99.99% SLA (with 2+ VMs in different zones)                   │
│   AWS equivalent: Availability Zones                              │
│                                                                       │
│ Security type:                                                      │
│   ● Trusted launch (recommended — Secure Boot + vTPM)           │
│   ○ Standard                                                      │
│   ○ Confidential (encrypted in-use memory — extra security)    │
│                                                                       │
│ Image: [Ubuntu Server 22.04 LTS - x64 Gen2 ▼]                    │
│ ⚡ Popular images:                                                   │
│   ├── Ubuntu Server 22.04 LTS / 24.04 LTS                      │
│   ├── Debian 12                                                   │
│   ├── Red Hat Enterprise Linux 8/9                               │
│   ├── Windows Server 2022 Datacenter                             │
│   ├── Windows 11 (for dev VMs)                                  │
│   └── [See all images] for marketplace (WordPress, SQL, etc.)  │
│                                                                       │
│ VM architecture: [x64 ▼]                                           │
│   ● x64 (Intel/AMD)                                               │
│   ○ Arm64 (Ampere — cheaper, use if your app supports ARM)     │
│                                                                       │
│ Run with Azure Spot discount:                                      │
│   □ (Unchecked by default)                                        │
│   ☑ Enables Spot pricing (up to 90% discount!)                  │
│   ⚠️ VM can be evicted anytime (for non-critical workloads)     │
│                                                                       │
│ Size: [Standard_D2s_v5 - 2 vcpus, 8 GiB memory ▼]              │
│ [See all sizes] → Filter by series, vCPU, memory                │
│                                                                       │
│ ── Administrator account ──                                         │
│                                                                       │
│ Authentication type:                                                │
│   ● SSH public key (Linux — RECOMMENDED)                         │
│   ○ Password                                                      │
│                                                                       │
│ Username: [azureuser]                                               │
│ ⚡ Don't use 'admin', 'root', 'test' (blocked by Azure)          │
│                                                                       │
│ SSH public key source:                                              │
│   ● Generate new key pair (Azure creates & you download)        │
│   ○ Use existing key stored in Azure                             │
│   ○ Use existing public key (paste your id_rsa.pub)            │
│                                                                       │
│ Key pair name: [vm-web-prod-01_key]                                │
│                                                                       │
│ ── Inbound port rules ──                                           │
│                                                                       │
│ Public inbound ports:                                               │
│   ● Allow selected ports                                          │
│   ○ None                                                          │
│                                                                       │
│ Select inbound ports: [SSH (22) ▼]                                │
│ ⚠️ Only for initial setup! Restrict to your IP later via NSG.    │
│ ⚡ For production: Select "None" and use Bastion or VPN.          │
│                                                                       │
│ [Next: Disks >]                                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: DISKS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── OS disk ──                                                       │
│                                                                       │
│ OS disk size: [Default (30 GiB) ▼]                                │
│ ⚡ Linux default: 30 GB, Windows default: 127 GB                   │
│   Can resize later (grow only, not shrink)                       │
│                                                                       │
│ OS disk type:                                                       │
│   ○ Premium SSD (P-series) ← RECOMMENDED for production        │
│   ○ Standard SSD (E-series) — Dev/test                          │
│   ○ Standard HDD (S-series) — Backup, infrequent access        │
│   ○ Premium SSD v2 — High IOPS/throughput (databases)          │
│   ○ Ultra Disk — Extreme performance (SAP, Oracle)             │
│                                                                       │
│ ┌───────────────┬──────────┬──────────┬─────────┬───────────────┐│
│ │ Disk Type     │ IOPS     │ Thput    │ Latency │ Use case      ││
│ ├───────────────┼──────────┼──────────┼─────────┼───────────────┤│
│ │ Standard HDD  │ 500      │ 60 MB/s  │ ~10ms   │ Backup, dev   ││
│ │ Standard SSD  │ 500-6000 │ 750 MB/s │ ~5ms    │ Dev/test, web ││
│ │ Premium SSD   │ 120-20K  │ 900 MB/s │ <1ms    │ Production ✅ ││
│ │ Premium SSDv2 │ Up to 80K│ 1200 MB/s│ <1ms    │ Databases     ││
│ │ Ultra Disk    │ Up to160K│ 4000 MB/s│ <0.5ms  │ SAP, Oracle   ││
│ └───────────────┴──────────┴──────────┴─────────┴───────────────┘│
│                                                                       │
│ Delete with VM:                                                      │
│   ☑ Delete OS disk when VM is deleted                             │
│                                                                       │
│ Key management:                                                      │
│   ● Platform-managed key (default — Microsoft manages)          │
│   ○ Customer-managed key (your Key Vault key)                   │
│   ○ Double encryption (platform + customer key)                 │
│                                                                       │
│ □ Enable Ultra Disk compatibility                                  │
│                                                                       │
│ ── Data disks ──                                                    │
│                                                                       │
│ [+ Create and attach a new disk]                                   │
│                                                                       │
│ Name: [disk-web-data-01]                                           │
│ Source type: ● None (empty)  ○ Snapshot  ○ Storage blob          │
│ Size: [64] GiB                                                      │
│ Type: [Premium SSD ▼]                                              │
│ □ Delete disk when VM is deleted                                   │
│ ⚡ UNCHECK this for data disks! Keep data disk even if VM deleted │
│                                                                       │
│ ⚡ Best practice:                                                    │
│   ├── OS disk: Keep small (30-64 GB), delete with VM            │
│   ├── Data disk: Separate, do NOT delete with VM                │
│   ├── Logs/temp: Can use local temp disk (free but ephemeral)  │
│   └── Database: Premium SSD v2 or Ultra Disk                    │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: NETWORKING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Network interface ──                                             │
│                                                                       │
│ Virtual network: [vnet-prod ▼]  [Create new]                      │
│ Subnet: [subnet-web (10.0.1.0/24) ▼]                             │
│                                                                       │
│ Public IP: [(new) vm-web-prod-01-ip ▼]                            │
│   ● Create new (Standard SKU, Static)                            │
│   ○ None (RECOMMENDED for production — use Load Balancer)       │
│   ⚡ For production: No public IP. Access via LB/Bastion/VPN.    │
│                                                                       │
│ NIC network security group:                                        │
│   ○ None                                                          │
│   ● Basic (creates simple NSG with selected ports)               │
│   ○ Advanced → [Select existing NSG: nsg-web-servers ▼]        │
│   ⚡ Production: Use Advanced → existing NSG (pre-configured)    │
│                                                                       │
│ Public inbound ports: [SSH (22) ▼]                                │
│                                                                       │
│ □ Delete public IP and NIC when VM is deleted: ☑                  │
│                                                                       │
│ ── Load balancing ──                                                │
│                                                                       │
│ □ Place this VM behind an existing load balancing solution?       │
│   Load balancer: [lb-web-prod ▼]                                 │
│   Backend pool: [bp-web-servers ▼]                                │
│   ⚡ Can add later. Usually done via VMSS, not individual VMs.   │
│                                                                       │
│ Accelerated networking: ☑ Enabled                                 │
│ ⚡ Free! Provides lower latency, higher throughput via SR-IOV.   │
│   Supported on most D/E/F series (4+ vCPUs).                    │
│                                                                       │
│ [Next: Management >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 4: MANAGEMENT                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Identity ──                                                      │
│ □ Enable system assigned managed identity                          │
│   ⚡ Allows VM to authenticate to Azure services (Key Vault, etc.)│
│   without storing credentials. RECOMMENDED.                      │
│                                                                       │
│ ── Azure AD (Entra ID) ──                                          │
│ □ Login with Azure AD: ☑                                          │
│   ⚡ SSH/RDP with your Azure AD credentials (no shared passwords!)│
│                                                                       │
│ ── Auto-shutdown ──                                                 │
│ □ Enable auto-shutdown: ☑                                         │
│   Shutdown time: [19:00]                                           │
│   Time zone: [(UTC+05:30) Chennai, Kolkata ▼]                    │
│   □ Send notification before shutdown                             │
│   ⚡ ENABLE for dev/test VMs to save costs!                       │
│   ⚠️ Don't enable for production (obviously)                     │
│                                                                       │
│ ── Backup ──                                                        │
│ □ Enable backup: ☑                                                 │
│   Recovery Services vault: [rsv-prod-india ▼]                    │
│   Backup policy: [DefaultPolicy ▼] (daily backup, 30 day retention)│
│   ⚡ ALWAYS enable backup for production VMs                      │
│                                                                       │
│ ── Guest OS updates (preview) ──                                   │
│ Patch orchestration: [Azure-orchestrated ▼]                       │
│   ● Azure-orchestrated (Azure manages patching schedule)        │
│   ○ AutomaticByPlatform                                          │
│   ○ Manual (you control patching)                                │
│   ○ ImageDefault                                                  │
│                                                                       │
│ □ Enable hotpatch (Windows only — patch without reboot)          │
│                                                                       │
│ ── Monitoring ──                                                    │
│ Boot diagnostics: ● Enable with managed storage account          │
│ ⚡ Enable! Helps debug boot failures (see console screenshot).   │
│                                                                       │
│ □ Enable OS guest diagnostics                                     │
│ □ Enable application health monitoring                           │
│                                                                       │
│ [Next: Advanced >]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 5: ADVANCED                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Extensions ──                                                    │
│ [+ Select an extension to install]                                 │
│ Popular extensions:                                                 │
│ ├── Custom Script Extension (run scripts on boot)               │
│ ├── Azure Monitor Agent (metrics & logs)                        │
│ ├── Network Watcher Agent                                        │
│ ├── Desired State Configuration (DSC)                           │
│ ├── Antimalware (Windows)                                       │
│ └── Key Vault extension (auto-rotate certificates)              │
│                                                                       │
│ ── Custom data (cloud-init) ──                                     │
│ [Paste cloud-init YAML or bash script]                            │
│ ⚡ Runs on FIRST boot only. For Linux: cloud-init format.        │
│ Example:                                                             │
│ #cloud-config                                                       │
│ packages:                                                            │
│   - nginx                                                           │
│   - docker.io                                                       │
│ runcmd:                                                              │
│   - systemctl start nginx                                          │
│                                                                       │
│ ── User data ──                                                     │
│ [Script that runs on every boot — passed to VM as metadata]      │
│ ⚡ Different from Custom data: User data persists and can be       │
│   read by the VM anytime via IMDS (metadata service).            │
│                                                                       │
│ ── Proximity placement group ──                                    │
│ [None ▼] or [ppg-web-india ▼]                                    │
│ ⚡ Co-locates VMs physically close (low latency between VMs).    │
│   Use for: Tightly coupled workloads (database + app tier).    │
│                                                                       │
│ ── Capacity reservation group ──                                   │
│ [None ▼]                                                           │
│ ⚡ Reserve compute capacity in a region (guarantees VMs available)│
│   Use for: DR readiness, compliance, guaranteed capacity.      │
│                                                                       │
│ [Next: Tags >]                                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 6: TAGS                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────┬──────────────┬─────────────────────────────────┐│
│ │ Name           │ Value        │ Resource                        ││
│ ├────────────────┼──────────────┼─────────────────────────────────┤│
│ │ environment    │ prod         │ All (VM, disk, NIC, IP)        ││
│ │ team           │ web          │ All                            ││
│ │ cost-center    │ CC-1234      │ All                            ││
│ │ application    │ myapp        │ All                            ││
│ │ managed-by     │ terraform    │ All                            ││
│ └────────────────┴──────────────┴─────────────────────────────────┘│
│                                                                       │
│ ⚡ Tags are CRITICAL for cost management and governance.           │
│   Tag ALL resources consistently.                                 │
│                                                                       │
│ [Review + create] → [Create]                                       │
│ ⚡ Download SSH private key when prompted (won't be available again)│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Availability Options

```
┌─────────────────────────────────────────────────────────────────────┐
│           AVAILABILITY: ZONES vs SETS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Availability Zones (RECOMMENDED):                                │
│    ├── 3 physically separate datacenters per region               │
│    ├── Independent power, cooling, networking                    │
│    ├── Deploy VMs across zones (Zone 1, 2, 3)                   │
│    ├── SLA: 99.99% (with VMs in 2+ zones)                       │
│    └── AWS equivalent: Availability Zones (same concept!)       │
│                                                                       │
│    ┌──────────────────────────────────────────────────────────┐  │
│    │ Region: Central India                                     │  │
│    │ ┌───────────┐  ┌───────────┐  ┌───────────┐            │  │
│    │ │  Zone 1   │  │  Zone 2   │  │  Zone 3   │            │  │
│    │ │ ┌──────┐  │  │ ┌──────┐  │  │ ┌──────┐  │            │  │
│    │ │ │VM-01 │  │  │ │VM-02 │  │  │ │VM-03 │  │            │  │
│    │ │ └──────┘  │  │ └──────┘  │  │ └──────┘  │            │  │
│    │ └───────────┘  └───────────┘  └───────────┘            │  │
│    │ If Zone 1 fails → VM-02 and VM-03 still running        │  │
│    └──────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. Availability Sets (legacy — still works):                       │
│    ├── Within SINGLE datacenter (not across zones!)              │
│    ├── Fault domains (FD): Separate racks (2-3 FDs)             │
│    ├── Update domains (UD): Reboot groups (5-20 UDs)           │
│    ├── SLA: 99.95%                                               │
│    ├── Use when: Availability Zones not available in region     │
│    └── ⚡ Prefer Availability Zones for new deployments          │
│                                                                       │
│ 3. Single VM (no redundancy):                                       │
│    ├── SLA: 99.9% (with Premium SSD)                            │
│    ├── SLA: 95% (with Standard HDD)                             │
│    └── Use for: Dev/test, non-critical workloads               │
│                                                                       │
│ Summary:                                                             │
│ ├── Production: 2+ VMs across Availability Zones → 99.99%      │
│ ├── Tight budget: Single VM + Premium SSD → 99.9%              │
│ └── VMSS (next chapter): Automatic across zones + autoscaling  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Spot VMs

```
┌─────────────────────────────────────────────────────────────────────┐
│           SPOT VMs                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Use Azure's spare capacity at up to 90% discount.           │
│ Catch: Azure can evict your VM with 30-second warning anytime.    │
│                                                                       │
│ ⚡ AWS equivalent: Spot Instances                                    │
│ ⚡ GCP equivalent: Spot VMs (formerly Preemptible)                  │
│                                                                       │
│ Configuration:                                                       │
│ Run with Azure Spot discount: ☑                                   │
│                                                                       │
│ Eviction type:                                                      │
│   ● Capacity only (evicted when Azure needs capacity)            │
│   ○ Price or capacity (evicted if price exceeds your max)       │
│                                                                       │
│ Eviction policy:                                                    │
│   ● Stop / Deallocate (VM stopped, can restart later)           │
│   ○ Delete (VM deleted permanently)                              │
│   ⚡ Use "Stop" for VMs you want to retry later                  │
│   ⚡ Use "Delete" for ephemeral workers                           │
│                                                                       │
│ Maximum price: [$0.05/hour] or [-1] (willing to pay up to PAYG) │
│ ⚡ Set to -1 to reduce eviction chance (pay market rate, still   │
│    much cheaper than regular PAYG most of the time).             │
│                                                                       │
│ Good for:                                                            │
│ ├── Batch processing / data analytics                            │
│ ├── CI/CD build agents                                           │
│ ├── Dev/test environments                                        │
│ ├── Stateless workers (process queue items)                     │
│ ├── ML training (checkpoint regularly)                          │
│ └── Rendering / HPC (embarrassingly parallel)                   │
│                                                                       │
│ NOT good for:                                                       │
│ ├── Production web servers                                       │
│ ├── Databases                                                    │
│ ├── Anything that can't handle sudden termination               │
│ └── SLA-bound workloads                                          │
│                                                                       │
│ Handling eviction:                                                   │
│ ├── Azure sends eviction event via Metadata Service (30s notice)│
│ ├── Poll: curl -H Metadata:true                                  │
│ │   "http://169.254.169.254/metadata/scheduledevents"          │
│ ├── Graceful shutdown: Save state, flush writes, deregister     │
│ └── For VMSS: Auto-replaces evicted Spot VMs                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Custom Images & Shared Image Gallery

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM IMAGES & GALLERY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Custom Image:                                                        │
│ ├── Snapshot of a configured VM (OS + software + config)         │
│ ├── Use as base for creating identical VMs                       │
│ ├── Pre-bake everything: OS patches, app, dependencies          │
│ └── Faster VM startup (no need to install on boot)              │
│                                                                       │
│ Creating a custom image:                                            │
│ 1. Create & configure a VM (install everything)                   │
│ 2. Generalize the VM:                                              │
│    Linux:  sudo waagent -deprovision+user -force                  │
│    Windows: sysprep /generalize /shutdown                         │
│ 3. Deallocate VM: az vm deallocate --name myvm -g myrg           │
│ 4. Generalize: az vm generalize --name myvm -g myrg              │
│ 5. Create image: az image create --name img-web-v1               │
│    --resource-group myrg --source myvm                           │
│                                                                       │
│ Azure Compute Gallery (formerly Shared Image Gallery):             │
│ ├── Centralized repository for custom images                    │
│ ├── Version management (1.0.0, 1.1.0, 2.0.0)                  │
│ ├── Replication across regions (image available globally)       │
│ ├── Share across subscriptions/tenants                          │
│ ├── RBAC access control                                         │
│ └── ⚡ Use Gallery for production image management               │
│                                                                       │
│ Gallery → Image definition → Image version:                       │
│ ├── Gallery: gal_company_images (org-wide)                      │
│ ├── Image definition: img-def-web-server                        │
│ │   ├── OS type: Linux                                          │
│ │   ├── Publisher: MyCompany                                    │
│ │   ├── Offer: WebServer                                        │
│ │   ├── SKU: Ubuntu2204-Nginx                                   │
│ │   └── Generation: V2                                          │
│ └── Image version: 1.2.0                                        │
│     ├── Source: managed image or snapshot                       │
│     ├── Replicate to: Central India, East US, West Europe      │
│     └── Replica count per region: 3 (for parallel VM creation) │
│                                                                       │
│ ⚡ AWS equivalent: AMI (Amazon Machine Image)                       │
│ ⚡ GCP equivalent: Custom Image + Image Family                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: VM Extensions

```
┌─────────────────────────────────────────────────────────────────────┐
│           VM EXTENSIONS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Small apps that run post-deployment on Azure VMs.           │
│ Think of them as "plugins" that configure/manage your VM.        │
│                                                                       │
│ Key extensions:                                                      │
│ ┌─────────────────────────┬──────────────────────────────────────┐│
│ │ Extension               │ Purpose                              ││
│ ├─────────────────────────┼──────────────────────────────────────┤│
│ │ CustomScript            │ Run bash/PowerShell on VM            ││
│ │ Azure Monitor Agent     │ Collect metrics & logs               ││
│ │ Dependency Agent        │ Map VM dependencies (Service Map)    ││
│ │ Network Watcher         │ Network diagnostics                  ││
│ │ AADSSHLoginForLinux     │ Azure AD login for SSH               ││
│ │ Key Vault (Linux)       │ Auto-rotate certs from Key Vault     ││
│ │ DSC (Desired State)     │ Configuration management             ││
│ │ Antimalware (Windows)   │ Microsoft Defender for VMs           ││
│ └─────────────────────────┴──────────────────────────────────────┘│
│                                                                       │
│ Custom Script Extension (most common):                              │
│ ├── Run a script on VM (install software, configure app)        │
│ ├── Script from: inline, Azure Storage, or GitHub URL           │
│ ├── Runs ONCE (not on every reboot — use cloud-init for that)  │
│ └── Timeout: 90 minutes                                          │
│                                                                       │
│ Example (az CLI):                                                   │
│ az vm extension set \                                               │
│   --resource-group rg-web-prod \                                   │
│   --vm-name vm-web-prod-01 \                                       │
│   --name CustomScript \                                             │
│   --publisher Microsoft.Azure.Extensions \                         │
│   --settings '{"commandToExecute":"apt-get update && apt-get install -y nginx"}'│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & Bicep Examples

### Terraform

```hcl
# Linux VM with Premium SSD, Availability Zone, Managed Identity
resource "azurerm_linux_virtual_machine" "web" {
  name                = "vm-web-prod-01"
  resource_group_name = azurerm_resource_group.web.name
  location            = azurerm_resource_group.web.location
  size                = "Standard_D2s_v5"
  zone                = "1"
  
  admin_username = "azureuser"
  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }
  
  network_interface_ids = [azurerm_network_interface.web.id]
  
  os_disk {
    name                 = "osdisk-web-prod-01"
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 30
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  boot_diagnostics {}  # Uses managed storage
  
  custom_data = base64encode(<<-EOF
    #cloud-config
    packages:
      - nginx
    runcmd:
      - systemctl enable nginx
      - systemctl start nginx
  EOF
  )
  
  tags = {
    environment = "prod"
    team        = "web"
  }
}

# Network interface
resource "azurerm_network_interface" "web" {
  name                = "nic-web-prod-01"
  location            = azurerm_resource_group.web.location
  resource_group_name = azurerm_resource_group.web.name
  
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.web.id
    private_ip_address_allocation = "Dynamic"
    # No public IP for production
  }
  
  accelerated_networking_enabled = true
}

# Data disk
resource "azurerm_managed_disk" "data" {
  name                 = "disk-web-data-01"
  location             = azurerm_resource_group.web.location
  resource_group_name  = azurerm_resource_group.web.name
  storage_account_type = "Premium_LRS"
  create_option        = "Empty"
  disk_size_gb         = 64
  zone                 = "1"
}

resource "azurerm_virtual_machine_data_disk_attachment" "data" {
  managed_disk_id    = azurerm_managed_disk.data.id
  virtual_machine_id = azurerm_linux_virtual_machine.web.id
  lun                = 0
  caching            = "ReadWrite"
}

# NSG
resource "azurerm_network_security_group" "web" {
  name                = "nsg-web-servers"
  location            = azurerm_resource_group.web.location
  resource_group_name = azurerm_resource_group.web.name
  
  security_rule {
    name                       = "AllowHTTP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# Spot VM (for workers/batch)
resource "azurerm_linux_virtual_machine" "worker" {
  name                = "vm-worker-spot-01"
  resource_group_name = azurerm_resource_group.web.name
  location            = azurerm_resource_group.web.location
  size                = "Standard_D4s_v5"
  
  priority        = "Spot"
  eviction_policy = "Deallocate"
  max_bid_price   = -1  # Pay up to PAYG price
  
  admin_username = "azureuser"
  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }
  
  network_interface_ids = [azurerm_network_interface.worker.id]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
}
```

### Bicep

```bicep
resource vm 'Microsoft.Compute/virtualMachines@2024-03-01' = {
  name: 'vm-web-prod-01'
  location: resourceGroup().location
  zones: ['1']
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    hardwareProfile: {
      vmSize: 'Standard_D2s_v5'
    }
    osProfile: {
      computerName: 'vm-web-prod-01'
      adminUsername: 'azureuser'
      linuxConfiguration: {
        disablePasswordAuthentication: true
        ssh: {
          publicKeys: [
            {
              path: '/home/azureuser/.ssh/authorized_keys'
              keyData: loadTextContent('~/.ssh/id_rsa.pub')
            }
          ]
        }
      }
    }
    storageProfile: {
      imageReference: {
        publisher: 'Canonical'
        offer: '0001-com-ubuntu-server-jammy'
        sku: '22_04-lts-gen2'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: 'Premium_LRS'
        }
        diskSizeGB: 30
      }
    }
    networkProfile: {
      networkInterfaces: [
        { id: nic.id }
      ]
    }
    diagnosticsProfile: {
      bootDiagnostics: {
        enabled: true
      }
    }
  }
  tags: {
    environment: 'prod'
    team: 'web'
  }
}
```

---

## Part 8: az CLI Reference

```bash
# Create VM (quick)
az vm create \
  --resource-group rg-web-prod \
  --name vm-web-prod-01 \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --zone 1 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --nsg nsg-web-servers \
  --vnet-name vnet-prod \
  --subnet subnet-web \
  --public-ip-address "" \
  --storage-sku Premium_LRS \
  --custom-data cloud-init.yaml \
  --tags environment=prod team=web

# List VMs
az vm list -g rg-web-prod \
  --query "[].{Name:name,Size:hardwareProfile.vmSize,Status:powerState}" \
  --output table

# Start/Stop/Restart
az vm start -g rg-web-prod -n vm-web-prod-01
az vm stop -g rg-web-prod -n vm-web-prod-01      # Stopped (still billed!)
az vm deallocate -g rg-web-prod -n vm-web-prod-01 # Deallocated (no charge)
az vm restart -g rg-web-prod -n vm-web-prod-01

# Resize VM
az vm resize -g rg-web-prod -n vm-web-prod-01 --size Standard_D4s_v5

# Add data disk
az vm disk attach \
  --resource-group rg-web-prod \
  --vm-name vm-web-prod-01 \
  --name disk-data-01 \
  --size-gb 64 \
  --sku Premium_LRS \
  --new

# Open port (add NSG rule)
az vm open-port -g rg-web-prod -n vm-web-prod-01 --port 80 --priority 100

# Run command on VM
az vm run-command invoke \
  -g rg-web-prod -n vm-web-prod-01 \
  --command-id RunShellScript \
  --scripts "apt-get update && apt-get install -y nginx"

# Create custom image
az vm deallocate -g rg-web-prod -n vm-web-prod-01
az vm generalize -g rg-web-prod -n vm-web-prod-01
az image create -g rg-web-prod --name img-web-v1 --source vm-web-prod-01

# Create Spot VM
az vm create \
  --resource-group rg-workers \
  --name vm-worker-spot-01 \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1 \
  --admin-username azureuser \
  --generate-ssh-keys

# SSH to VM (via Bastion)
az network bastion ssh \
  --name bastion-prod \
  --resource-group rg-networking \
  --target-resource-id /subscriptions/.../vm-web-prod-01 \
  --auth-type ssh-key \
  --username azureuser \
  --ssh-key ~/.ssh/id_rsa

# Enable auto-shutdown
az vm auto-shutdown \
  -g rg-web-prod -n vm-web-prod-01 \
  --time 1900 \
  --timezone "India Standard Time"

# List available sizes in region
az vm list-sizes --location centralindia --output table

# List available images
az vm image list --offer Ubuntu --publisher Canonical --sku 22_04 --all --output table
```

---

## Part 9: Real-World Patterns

### Startup

```
Setup: 1-2 VMs for web app

VM: vm-web-prod-01
├── Size: B2s (2 vCPU, 4 GB) — burstable, cheap
├── Image: Ubuntu 22.04 LTS
├── Disk: Premium SSD 30 GB (OS) — gets 99.9% SLA
├── Zone: Zone 1 (single VM, no HA yet)
├── Public IP: Yes (direct access, SSL via Let's Encrypt)
├── NSG: Allow 80, 443 from anywhere. SSH from your IP only.
├── Auto-shutdown: 7 PM (if dev VM)
├── Backup: Enabled, daily, 30-day retention
├── Managed identity: Yes (access Key Vault for secrets)
└── Custom data: cloud-init installs app stack

When growing (add HA):
├── Add second VM in different zone
├── Put behind Azure Load Balancer
├── Remove public IPs from VMs
└── Or: Move to App Service (less ops overhead)

Cost: ~$35/month (B2s) + ~$5 disk + $5 backup = ~$45/month
```

### Mid-Size

```
Architecture: Multiple VMs with Load Balancer

Web tier:
├── 3× Standard_D2s_v5 (across 3 zones)
├── Custom image (pre-baked with app + nginx)
├── Behind Azure Load Balancer (Standard)
├── No public IPs on VMs
├── NSG: Allow 80/443 from LB subnet, SSH from Bastion only
├── Managed identity → Key Vault for secrets
└── Azure Monitor Agent → Log Analytics

App tier:
├── 2× Standard_D4s_v5 (Zone 1, Zone 2)
├── Behind Internal Load Balancer
├── VNet integration with web tier
└── Access: RDS, Redis, Storage via private endpoints

Workers:
├── 2× Standard_D4s_v5 (Spot VMs — 60-90% cheaper)
├── Process jobs from Service Bus queue
├── Eviction handling: checkpoint + retry
└── Can tolerate interruption

Deployment:
├── Custom images built in CI (Packer)
├── Deploy: Create new VMs from new image → swap in LB → drain old
├── Or: Custom Script Extension to pull latest code
├── Backup: All VMs, daily, 30-day retention
└── Patching: Azure Update Manager (scheduled weekly)

Monitoring:
├── Azure Monitor: CPU, memory, disk, network
├── VM Insights: Performance + dependency mapping
├── Alerts: CPU > 80%, disk > 90%, VM unresponsive
├── Log Analytics: Syslog, app logs, security logs
└── Boot diagnostics: Debug boot failures

Cost: ~$500-800/month
```

### Enterprise

```
Architecture: Multi-region, highly available, governed

Compute:
├── Region 1 (Central India): Primary
│   ├── Web: 6× D4s_v5 (3 zones × 2 VMs) — or use VMSS (Ch16)
│   ├── App: 4× D8s_v5 (2 zones)
│   ├── DB: 2× E8s_v5 (availability set — co-locate with data)
│   └── Workers: 4× D4s_v5 Spot
├── Region 2 (West Europe): DR/Secondary
│   ├── Web: 2× D4s_v5 (standby, or active-active via TM)
│   ├── App: 2× D8s_v5
│   └── DB: Replica (Azure Site Recovery)
└── Global: Traffic Manager (performance routing between regions)

Images:
├── Azure Compute Gallery (shared across subscriptions)
├── Image pipeline: Packer → Gallery → Replicate to regions
├── Monthly golden image: OS patches + security hardening
├── Image versions: 2024.05.01, 2024.06.01, etc.
└── RBAC: Only CI/CD can push images, teams can only read

Security:
├── No public IPs (all access via Azure Bastion or VPN)
├── Azure AD login for SSH (no shared keys)
├── Managed identity (no secrets on VMs)
├── Disk encryption: Customer-managed keys (Key Vault)
├── Confidential VMs for PII processing workloads
├── Just-in-time VM access (open SSH port only when needed)
├── Microsoft Defender for Servers (threat detection)
├── OS hardening: CIS benchmark compliance
└── NSG flow logs → Log Analytics (audit trail)

Governance:
├── Azure Policy: Enforce tags, approved VM sizes, regions
├── Policy: Deny VMs without managed identity
├── Policy: Require Premium SSD for production
├── Policy: Auto-deploy Azure Monitor Agent
├── Cost management: Reservations (1-year: 40%, 3-year: 60% savings)
└── Savings Plans: Commit to $/hour (flexible across sizes)

DR:
├── Azure Site Recovery: Replicate VMs to DR region
├── RPO: 5 minutes (continuous replication)
├── RTO: 30 minutes (automated failover)
├── DR drills: Quarterly test failover
└── Backup: Daily (30-day), weekly (12-week), monthly (12-month)

Cost: $5,000-15,000/month (with reservations saving 40-60%)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | IaaS compute — full control OS/software |
| Series | B (burstable), D (general), E (memory), F (compute), N (GPU) |
| Naming | Standard_D4as_v5 = D-series, 4 vCPU, AMD, Premium SSD capable, v5 |
| OS disk | Premium SSD recommended for production |
| Data disk | Separate from OS, do NOT delete with VM |
| Availability Zones | 3 separate DCs per region → 99.99% SLA |
| Availability Sets | Same DC, separate racks → 99.95% SLA (legacy) |
| Spot VMs | Up to 90% discount, can be evicted anytime |
| Custom images | Pre-baked VM snapshots (like AMI) |
| Compute Gallery | Versioned image repo, cross-region replication |
| Extensions | Post-deploy plugins (Custom Script, Monitor, etc.) |
| Managed Identity | VM authenticates to Azure services (no passwords) |
| SLA (single VM) | 99.9% with Premium SSD |
| AWS equivalent | EC2 |
| GCP equivalent | Compute Engine |

---

## What's Next?

In the next chapter, we'll cover Azure VM Scale Sets — automatic scaling of identical VMs.

→ Next: [Chapter 16: VM Scale Sets](16-vmss.md)

---

*Last Updated: May 2026*
