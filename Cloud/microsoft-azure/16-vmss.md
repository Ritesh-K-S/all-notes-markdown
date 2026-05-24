# Chapter 16: Azure VM Scale Sets (VMSS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: VMSS Fundamentals](#part-1-vmss-fundamentals)
- [Part 2: Creating a VMSS (Full Portal Walkthrough)](#part-2-creating-a-vmss-full-portal-walkthrough)
- [Part 3: Scaling Policies (Deep Dive)](#part-3-scaling-policies-deep-dive)
- [Part 4: Upgrade Policies](#part-4-upgrade-policies)
- [Part 5: Automatic Instance Repair](#part-5-automatic-instance-repair)
- [Part 6: Terraform & Bicep Examples](#part-6-terraform--bicep-examples)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Virtual Machine Scale Sets (VMSS) let you create and manage a group of identical, load-balanced VMs. The number of VM instances can automatically increase or decrease in response to demand or a defined schedule. VMSS provides high availability, automatic scaling, and centralized management for your VM fleet.

```
What you'll learn:
├── VMSS Fundamentals
│   ├── What & why (identical VM fleet, auto-managed)
│   ├── VMSS vs individual VMs vs App Service
│   ├── Orchestration modes (Uniform vs Flexible)
│   └── How it works with Load Balancers
├── Creating a VMSS (Full Portal Walkthrough)
│   ├── Basics (name, region, zones, image, size)
│   ├── Spot instances
│   ├── Disks
│   ├── Networking (VNet, NSG, LB, NAT)
│   ├── Scaling (manual, auto-scale rules, scheduled)
│   ├── Management (upgrades, health, extensions)
│   ├── Health (app health, auto-repair)
│   └── Advanced (custom data, proximity groups)
├── Scaling Policies
│   ├── Manual scaling
│   ├── Custom autoscale (rules-based)
│   ├── Metric-based (CPU, memory, queue depth)
│   ├── Schedule-based
│   └── Predictive autoscale
├── Upgrade Policies
│   ├── Automatic (immediately apply)
│   ├── Rolling (gradual with surge/unavailable)
│   ├── Manual (you control)
│   └── MaxSurge rolling upgrade
├── Instance Repair (Automatic)
├── Orchestration Modes (Uniform vs Flexible)
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: VMSS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           VMSS CONCEPT                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without VMSS:                                                        │
│ ├── Create 5 VMs manually, one by one                            │
│ ├── Configure each separately                                    │
│ ├── Monitor each for failures, replace manually                 │
│ ├── Scale: Create/delete VMs manually                           │
│ └── Updates: SSH into each VM, update code/config               │
│                                                                       │
│ With VMSS:                                                           │
│ ├── Define VM model ONCE (image, size, config)                  │
│ ├── VMSS creates identical VMs from that model                  │
│ ├── Auto-heals: Detects failures, replaces unhealthy VMs       │
│ ├── Auto-scales: Adds/removes VMs based on metrics             │
│ └── Rolling updates: Deploy new model without downtime          │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ VM Scale Set: vmss-web-prod                                  │  │
│ │ Model: Ubuntu 22.04 + nginx, D2s_v5, Premium SSD           │  │
│ │                                                              │  │
│ │ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                        │  │
│ │ │VM 0│ │VM 1│ │VM 2│ │VM 3│ │VM 4│  ← All identical       │  │
│ │ └──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘                        │  │
│ │    │      │      │      │      │                            │  │
│ │    └──────┼──────┼──────┼──────┘                            │  │
│ │           │      │      │                                    │  │
│ │    ┌──────┴──────┴──────┴──────┐                            │  │
│ │    │    Azure Load Balancer     │                            │  │
│ │    └────────────────────────────┘                            │  │
│ │                                                              │  │
│ │ Auto-scale: 2 min → 10 max (based on CPU > 70%)           │  │
│ │ Spread across: Zone 1, Zone 2, Zone 3                      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ AWS equivalent: Auto Scaling Group + Launch Template             │
│ ⚡ GCP equivalent: Managed Instance Group + Instance Template      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           ORCHESTRATION MODES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌───────────────────┬─────────────────────────────────────────────┐│
│ │ Uniform (legacy)  │ Flexible (recommended) ✅                  ││
│ ├───────────────────┼─────────────────────────────────────────────┤│
│ │ All VMs identical │ Mix VM sizes in same VMSS               ││
│ │ VMSS controls all │ Can add existing VMs                    ││
│ │ VM model required │ Multiple VM profiles                    ││
│ │ 1000 VM limit     │ 1000 VM limit                            ││
│ │ Zones: Spread     │ Zones: Spread (max spreading)           ││
│ │ Upgrade policies  │ Auto, Rolling, Manual                   ││
│ │ Networking: Basic │ Networking: Standard LB ✅               ││
│ │ Overprovisioning  │ No overprovisioning                     ││
│ └───────────────────┴─────────────────────────────────────────────┘│
│                                                                       │
│ ⚡ Flexible mode recommended for most new VMSS deployments.        │
│   Supports Spot + regular mix, different VM sizes, better AZ     │
│   spreading. Uniform is legacy but still widely used.            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a VMSS (Full Portal Walkthrough)

```
Console → Virtual machine scale sets → Create

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-web-prod ▼]                                    │
│                                                                       │
│ Virtual machine scale set name: [vmss-web-prod]                   │
│ ⚡ Naming: vmss-{purpose}-{env}                                     │
│                                                                       │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Availability zones: ☑ Zone 1  ☑ Zone 2  ☑ Zone 3                 │
│ ⚡ Select ALL zones for maximum availability (99.99% SLA).        │
│                                                                       │
│ Zone balance: ☑ Strictly enforce even zone distribution          │
│ ⚡ Even: Each zone gets equal VMs. If a zone can't fit → fail.   │
│   Unbalanced: Best-effort distribution (more flexible).         │
│                                                                       │
│ Orchestration mode:                                                 │
│   ● Flexible (recommended)                                       │
│   ○ Uniform                                                       │
│                                                                       │
│ Security type: ● Trusted launch virtual machines                 │
│                                                                       │
│ Image: [Ubuntu Server 22.04 LTS - Gen2 ▼]                        │
│ [See all images] → Marketplace                                    │
│ Or: [My images] → Your custom image from Compute Gallery        │
│                                                                       │
│ VM architecture: [x64 ▼]                                           │
│                                                                       │
│ Run with Azure Spot discount:                                      │
│   □ (Unchecked — covered below in detail)                        │
│                                                                       │
│ Size: [Standard_D2s_v5 - 2 vcpus, 8 GiB memory ▼]              │
│ [See all sizes]                                                     │
│                                                                       │
│ ── Administrator account ──                                         │
│ Authentication type: ● SSH public key                             │
│ Username: [azureuser]                                               │
│ SSH public key source: [Generate new key pair ▼]                 │
│ Key pair name: [vmss-web-prod_key]                                │
│                                                                       │
│ [Next: Spot >]                                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: SPOT                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ☐ Run with Azure Spot discount                                     │
│                                                                       │
│ (If checked:)                                                       │
│ Eviction type: ● Capacity only  ○ Price or capacity             │
│ Eviction policy: ● Deallocate  ○ Delete                          │
│ Max price/hr: [$-1] (-1 = up to pay-as-you-go price)            │
│                                                                       │
│ ⚡ For VMSS, Spot is powerful:                                      │
│   ├── VMSS auto-replaces evicted Spot VMs                       │
│   ├── Mix: Spot + regular VMs in same VMSS (Flexible mode)     │
│   ├── Base: 3 regular (guaranteed) + burst: Spot (cheap)       │
│   └── 60-90% savings on scaling/burst instances                │
│                                                                       │
│ [Next: Disks >]                                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: DISKS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ OS disk type: [Premium SSD ▼] ← Recommended for production      │
│ OS disk size: [Default (30 GiB)]                                  │
│ Encryption type: [Default — platform-managed key ▼]              │
│                                                                       │
│ ── Data disks ──                                                    │
│ [+ Create and attach a new disk]                                   │
│ Size: [64] GiB                                                      │
│ Type: [Premium SSD ▼]                                              │
│ ⚡ Data disks are per-instance. Each VM gets its own.             │
│ ⚠️ Data disks are ephemeral with VMSS! When instance is deleted  │
│    → data disk deleted too (unless using Flexible mode + keep)  │
│    For persistent data: Use Azure Files, Blob, managed databases│
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 4: NETWORKING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Virtual network: [vnet-prod ▼]                                     │
│ Subnet: [subnet-web (10.0.1.0/24) ▼]                             │
│                                                                       │
│ Network interface:                                                  │
│ ├── NIC network security group: [Advanced ▼]                    │
│ │   NSG: [nsg-vmss-web ▼]                                       │
│ ├── Public IP address: [Disabled ▼] ← No public IP per VM     │
│ │   ⚡ Production: No public IP. Access via Load Balancer.      │
│ └── Accelerated networking: ☑ Enabled                           │
│                                                                       │
│ ── Load balancing ──                                                │
│                                                                       │
│ Load balancing options:                                             │
│   ● Azure load balancer                                           │
│   ○ Application gateway                                          │
│   ○ None                                                          │
│                                                                       │
│ Load balancer: [lb-web-prod ▼]  [Create new]                     │
│                                                                       │
│ [Create new load balancer:]                                        │
│ Name: [lb-web-prod]                                                 │
│ Type: ● Public  ○ Internal                                       │
│ SKU: Standard (always use Standard for VMSS!)                    │
│ Frontend IP: [Create new] → [pip-lb-web-prod]                   │
│ Backend pool: [bp-web-servers]                                    │
│                                                                       │
│ ⚡ VMSS auto-registers/deregisters instances in backend pool.     │
│   New instance → added to LB. Instance deleted → removed.      │
│                                                                       │
│ Or: Application Gateway (L7 — path-based routing, WAF):         │
│ Application Gateway: [appgw-web-prod ▼]                          │
│ Backend pool: [bp-appgw-web]                                      │
│                                                                       │
│ ── Inbound NAT rule (for SSH access) ──                            │
│ ☐ Use NAT rules to access each VM instance                       │
│ ⚡ Maps LB frontend port range → individual VM SSH ports         │
│   Example: LB:50001 → VM0:22, LB:50002 → VM1:22                │
│   ⚠️ For dev only. Production: Use Azure Bastion.               │
│                                                                       │
│ [Next: Scaling >]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 5: SCALING                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scaling mode:                                                       │
│   ○ Manual scale (fixed instance count)                           │
│   ● Custom autoscale (rules-based — RECOMMENDED)                │
│                                                                       │
│ ═══ MANUAL SCALE ═══                                               │
│ Instance count: [3]                                                 │
│ ⚡ Fixed size. No auto-scaling. Simple.                            │
│                                                                       │
│ ═══ CUSTOM AUTOSCALE ═══                                           │
│                                                                       │
│ Autoscale setting name: [as-web-prod]                             │
│                                                                       │
│ Default rule:                                                       │
│ Scale mode: ● Scale based on a metric                             │
│             ○ Scale to a specific instance count                 │
│                                                                       │
│ Instance limits:                                                    │
│ Minimum: [2]                                                        │
│ Maximum: [10]                                                       │
│ Default: [3] (used when metrics unavailable)                      │
│ ⚡ Default is fallback count when metrics can't be read.          │
│                                                                       │
│ ── Scale-out rule (add VMs when busy) ──                           │
│ [+ Add a rule]                                                      │
│                                                                       │
│ Metric source: [Current resource (vmss-web-prod) ▼]              │
│ ⚡ Can also scale based on:                                        │
│   ├── Other Azure resource (Service Bus queue depth)            │
│   ├── Application Insights (custom app metrics)                │
│   └── Storage queue (messages in queue)                         │
│                                                                       │
│ Metric name: [Percentage CPU ▼]                                   │
│   ○ Percentage CPU                                                │
│   ○ Network In Total                                              │
│   ○ Network Out Total                                             │
│   ○ Disk Read Bytes                                               │
│   ○ Available Memory Bytes (via guest agent)                    │
│   ○ ... many more                                                 │
│                                                                       │
│ Time aggregation: [Average ▼]                                     │
│ Operator: [Greater than ▼]                                        │
│ Metric threshold: [70]                                              │
│ Duration: [10] minutes                                              │
│ ⚡ "When average CPU > 70% for 10 minutes → scale out"           │
│                                                                       │
│ Operation: [Increase count by ▼]                                  │
│ Instance count: [2]                                                 │
│ Cool down: [5] minutes (wait before evaluating again)            │
│ ⚡ Increase by 2 instances (or: percentage, set to exact count)  │
│                                                                       │
│ ── Scale-in rule (remove VMs when idle) ──                         │
│ [+ Add a rule]                                                      │
│                                                                       │
│ Metric: Percentage CPU                                              │
│ Operator: Less than                                                 │
│ Threshold: [30]                                                     │
│ Duration: [10] minutes                                              │
│ ⚡ "When average CPU < 30% for 10 minutes → scale in"            │
│                                                                       │
│ Operation: Decrease count by [1]                                   │
│ Cool down: [10] minutes                                             │
│ ⚡ Scale-in cooldown longer than scale-out (conservative)        │
│                                                                       │
│ ── Schedule-based scaling ──                                        │
│ [+ Add a scale condition]                                          │
│                                                                       │
│ Scale mode: Scale to a specific instance count                    │
│ Instance count: [8]                                                 │
│ Schedule:                                                            │
│   Repeat specific days: ☑ Mon ☑ Tue ☑ Wed ☑ Thu ☑ Fri           │
│   Start time: 08:00                                                │
│   End time: 20:00                                                  │
│   Timezone: (UTC+05:30) Chennai, Kolkata                         │
│                                                                       │
│ ⚡ "Business hours: min 8 instances. Off-hours: min 2."          │
│   Add another condition for weekends (min 2).                    │
│                                                                       │
│ ── Predictive autoscale (preview) ──                               │
│ ☐ Enable predictive autoscale                                     │
│ ⚡ Uses ML to predict load and pre-scale.                         │
│   Preview feature — enable for production when GA.              │
│                                                                       │
│ [Next: Management >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 6: MANAGEMENT                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Upgrade policy ──                                                │
│                                                                       │
│ Upgrade mode:                                                       │
│   ○ Automatic (apply changes to ALL instances immediately)      │
│   ● Rolling (gradual, batch by batch — RECOMMENDED for prod)   │
│   ○ Manual (you trigger upgrades per instance)                  │
│                                                                       │
│ ═══ ROLLING UPGRADE POLICY ═══                                     │
│                                                                       │
│ Max batch percentage: [20]%                                        │
│ ⚡ Upgrade 20% of instances at a time (5 VMs → 1 per batch).    │
│                                                                       │
│ Max unhealthy instance percentage: [20]%                          │
│ ⚡ If > 20% of all instances are unhealthy → pause upgrade.     │
│                                                                       │
│ Max unhealthy upgraded instance percentage: [20]%                 │
│ ⚡ If > 20% of upgraded instances unhealthy → pause.            │
│                                                                       │
│ Pause time between batches: [0] seconds                           │
│ ⚡ Wait between batches. Set 60-120s for monitoring.             │
│                                                                       │
│ ☑ Enable cross-zone upgrade                                       │
│ ⚡ Upgrade across zones. Without this: Upgrade zone by zone.     │
│                                                                       │
│ Prioritize unhealthy instances: ☑                                 │
│ ⚡ Upgrade unhealthy instances first (they need new model anyway)│
│                                                                       │
│ ☑ Enable MaxSurge                                                  │
│ ⚡ Create NEW instances with new model, then delete old ones.     │
│   Without MaxSurge: Delete old instance → create new (downtime).│
│   With MaxSurge: Create new → verify healthy → delete old.     │
│   ⚡ RECOMMENDED for zero-downtime updates.                      │
│                                                                       │
│ ── Identity ──                                                      │
│ ☑ Enable system assigned managed identity                        │
│                                                                       │
│ ── Health monitoring ──                                             │
│ ☑ Enable application health monitoring                           │
│ Health probe:                                                       │
│   ● Use a load balancer health probe                             │
│   ○ Use an application health extension                         │
│                                                                       │
│ Load balancer health probe: [lb-probe-http-80 ▼]                │
│ ⚡ VMSS uses the LB health probe to determine instance health.   │
│                                                                       │
│ ── Automatic repair ──                                              │
│ ☑ Enable automatic repairs                                        │
│ Grace period: [30] minutes                                         │
│ Repair action: ● Replace  ○ Reimage  ○ Restart                  │
│                                                                       │
│ ⚡ If instance unhealthy → automatically replace after grace     │
│   period. Grace period: How long to wait before considering     │
│   the instance truly unhealthy (avoids killing during startup). │
│                                                                       │
│ ⚡ Repair actions:                                                   │
│   ├── Replace: Delete + create new (cleanest, default) ✅       │
│   ├── Reimage: Reinstall OS from image (keeps data disk)       │
│   └── Restart: Simple restart (quick but may not fix issue)    │
│                                                                       │
│ ── Boot diagnostics ──                                              │
│ ☑ Enable with managed storage account                            │
│                                                                       │
│ [Next: Advanced >]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 7: ADVANCED                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Spreading algorithm ──                                           │
│                                                                       │
│ Spreading: [Max spreading (recommended) ▼]                       │
│   ● Max spreading (spreads across as many FDs as possible)      │
│   ○ Fixed spreading: [5] fault domains                           │
│   ⚡ Max spreading provides best availability.                    │
│                                                                       │
│ ── Custom data (cloud-init) ──                                     │
│ [Paste cloud-init YAML]                                            │
│ #cloud-config                                                       │
│ packages:                                                            │
│   - nginx                                                           │
│   - docker.io                                                       │
│ runcmd:                                                              │
│   - systemctl start nginx                                          │
│   - systemctl enable nginx                                         │
│                                                                       │
│ ── Extensions ──                                                    │
│ [+ Add an extension]                                                │
│ ├── Custom Script (run post-deploy script on each VM)           │
│ ├── Azure Monitor Agent                                          │
│ ├── Application Health (for auto-repair without LB)             │
│ └── Key Vault (certificate management)                          │
│                                                                       │
│ ── Proximity placement group ──                                    │
│ [None ▼]                                                           │
│                                                                       │
│ ── Overprovisioning (Uniform mode only) ──                         │
│ ☐ Enable overprovisioning                                         │
│ ⚡ Creates extra VMs during scaling, keeps only needed amount.    │
│   Faster scaling (VMs ready sooner) but uses more capacity.     │
│   Not available in Flexible mode.                                │
│                                                                       │
│ [Next: Tags > ] → [Review + create] → [Create]                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Scaling Policies (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTOSCALE RULES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Autoscale uses metric rules: "When X happens → do Y"             │
│                                                                       │
│ Metric-based rules:                                                 │
│ ┌────────────────────────────┬────────────────────────────────┐   │
│ │ Metric                    │ Best for                        │   │
│ ├────────────────────────────┼────────────────────────────────┤   │
│ │ Percentage CPU             │ CPU-bound workloads (default) │   │
│ │ Available Memory Bytes     │ Memory-intensive apps          │   │
│ │ Network In/Out Total       │ Network-intensive apps         │   │
│ │ Disk Read/Write Bytes      │ I/O-intensive apps             │   │
│ │ OS Disk Queue Depth        │ Disk bottleneck detection     │   │
│ │ Inbound/Outbound Flows     │ Connection-heavy apps          │   │
│ │ Service Bus Queue Length   │ Queue-based workers            │   │
│ │ App Insights Custom Metric │ Custom app metrics             │   │
│ └────────────────────────────┴────────────────────────────────┘   │
│                                                                       │
│ Best practice: ALWAYS create BOTH scale-out AND scale-in rules!  │
│ ├── Scale-out: CPU > 70% for 10 min → add 2 instances         │
│ ├── Scale-in: CPU < 30% for 10 min → remove 1 instance        │
│ ├── Scale-out cooldown: 5 min (react quickly to load)          │
│ └── Scale-in cooldown: 10 min (conservative, avoid thrashing)  │
│                                                                       │
│ ⚡ Scale-out threshold should NOT equal scale-in threshold!       │
│   Bad: Scale-out > 70%, Scale-in < 70% → FLAPPING!             │
│   Good: Scale-out > 70%, Scale-in < 30% → Stable!             │
│   The gap between thresholds is the "dead zone" (no action).   │
│                                                                       │
│ Schedule-based:                                                     │
│ ├── Business hours (Mon-Fri 8AM-8PM): min 6 instances          │
│ ├── Off-hours (nights, weekends): min 2 instances              │
│ ├── Sale/event days: min 20 instances (pre-scale!)             │
│ └── ⚡ Schedule + metric rules combine (highest count wins)      │
│                                                                       │
│ External metrics (queue-based workers):                            │
│ ├── Service Bus queue: When messages > 1000 → add 2 workers   │
│ ├── Storage queue: When messages > 500 → add 1 worker          │
│ └── Application Insights: When req/sec > 5000 → scale out    │
│                                                                       │
│ Scale-in policy (Uniform mode):                                    │
│ ├── Default: Balanced across zones, then highest instance ID   │
│ ├── NewestVM: Remove newest first                               │
│ ├── OldestVM: Remove oldest first                               │
│ └── ⚡ Default is best for zone-balanced deployments             │
│                                                                       │
│ Notifications:                                                      │
│ ├── Email: Notify on scale-out/scale-in events                  │
│ ├── Webhook: Call URL on scaling events                         │
│ └── Action group: Full alert pipeline (email, SMS, Logic App)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Upgrade Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           UPGRADE POLICIES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When you update the VMSS model (new image, config change), how   │
│ do existing instances get updated?                                 │
│                                                                       │
│ 1. Manual:                                                          │
│    ├── You explicitly trigger upgrade per instance                │
│    ├── az vmss update-instances --instance-ids "0" "1" "2"      │
│    ├── Full control but manual work                              │
│    └── Use for: Testing, careful production updates             │
│                                                                       │
│ 2. Automatic:                                                       │
│    ├── ALL instances updated immediately when model changes      │
│    ├── Fast but risky (all at once, no control)                 │
│    ├── ⚡ Only use for dev/test or stateless workers             │
│    └── ⚠️ Not recommended for production web tiers              │
│                                                                       │
│ 3. Rolling (RECOMMENDED for production):                           │
│    ├── Batch by batch, configurable pace                        │
│    ├── Max batch %: 20% (upgrade 20% at a time)                │
│    ├── Pause between batches: 60s (monitoring window)           │
│    ├── Unhealthy thresholds: Pause if too many fail            │
│    └── ⚡ Zero-downtime with proper configuration                │
│                                                                       │
│    Without MaxSurge:                                               │
│    [VM0-old] delete → [VM0-new] create → verify healthy         │
│    [VM1-old] delete → [VM1-new] create → verify healthy         │
│    ⚠️ Capacity temporarily reduced during upgrade!              │
│                                                                       │
│    With MaxSurge ✅:                                               │
│    [VM5-new] create → verify healthy → [VM0-old] delete        │
│    [VM6-new] create → verify healthy → [VM1-old] delete        │
│    ⚡ Capacity NEVER drops below current level!                  │
│    ⚡ Always use MaxSurge for production!                        │
│                                                                       │
│ Rolling upgrade flow:                                               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ VMSS: 5 instances, Rolling, 20% batch, MaxSurge enabled     │  │
│ │                                                              │  │
│ │ Step 1: Create VM5-new (surge)                              │  │
│ │         [v1][v1][v1][v1][v1][v2-new] → 6 total             │  │
│ │                                                              │  │
│ │ Step 2: VM5-new healthy → Delete VM0-old                   │  │
│ │         [v1][v1][v1][v1][v2] → 5 total                     │  │
│ │                                                              │  │
│ │ Step 3: Create VM6-new → healthy → Delete VM1-old          │  │
│ │         [v1][v1][v1][v2][v2] → 5 total                     │  │
│ │                                                              │  │
│ │ ... repeat until all upgraded ...                           │  │
│ │                                                              │  │
│ │ Final: [v2][v2][v2][v2][v2] → all upgraded!                │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Automatic Instance Repair

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTOMATIC INSTANCE REPAIR                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Automatically detect and replace unhealthy instances.        │
│                                                                       │
│ Flow:                                                                │
│ Health probe → Instance unhealthy → Wait grace period →          │
│ Repair action (Replace/Reimage/Restart)                           │
│                                                                       │
│ Health sources:                                                      │
│ ├── Load Balancer health probe (if VMSS has LB)                 │
│ │   LB probes → /health → 200 OK = healthy, else = unhealthy  │
│ ├── Application Health Extension (if no LB)                     │
│ │   Extension installed on VM → probes local app endpoint      │
│ └── ⚡ At least one health source required for auto-repair!      │
│                                                                       │
│ Application Health Extension (for VMs without LB):                 │
│ ├── Install extension on VMSS                                    │
│ ├── Probes local app: http://localhost:8080/health              │
│ ├── Reports health status to VMSS                               │
│ └── Used when VMs process queues (no LB in front)              │
│                                                                       │
│ Grace period:                                                        │
│ ├── Time to wait after instance creation before health checks   │
│ ├── MUST be >= boot time + app startup time                     │
│ ├── Too short: Healthy VMs killed before they start!           │
│ ├── Recommended: 30 min for typical apps (be generous)         │
│ └── Range: 30 min - 90 min                                      │
│                                                                       │
│ Repair actions:                                                      │
│ ├── Replace: Delete + create new (default, cleanest)           │
│ ├── Reimage: Reinstall OS, keep data disk                      │
│ └── Restart: Quick restart (may not fix root cause)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & Bicep Examples

### Terraform

```hcl
# Linux VMSS (Flexible orchestration)
resource "azurerm_orchestrated_virtual_machine_scale_set" "web" {
  name                = "vmss-web-prod"
  location            = azurerm_resource_group.web.location
  resource_group_name = azurerm_resource_group.web.name
  
  sku_name                    = "Standard_D2s_v5"
  instances                   = 3
  platform_fault_domain_count = 1  # For zone-spanning
  
  zones = ["1", "2", "3"]
  zone_balance = true
  
  os_profile {
    linux_configuration {
      admin_username = "azureuser"
      admin_ssh_key {
        username   = "azureuser"
        public_key = file("~/.ssh/id_rsa.pub")
      }
      disable_password_authentication = true
    }
    
    custom_data = base64encode(<<-EOF
      #cloud-config
      packages:
        - nginx
      runcmd:
        - systemctl enable nginx
        - systemctl start nginx
    EOF
    )
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
  
  os_disk {
    storage_account_type = "Premium_LRS"
    caching              = "ReadWrite"
    disk_size_gb         = 30
  }
  
  network_interface {
    name    = "nic-vmss-web"
    primary = true
    
    ip_configuration {
      name      = "internal"
      primary   = true
      subnet_id = azurerm_subnet.web.id
      
      load_balancer_backend_address_pool_ids = [
        azurerm_lb_backend_address_pool.web.id
      ]
    }
    
    enable_accelerated_networking = true
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  boot_diagnostics {}
  
  extension {
    name                 = "HealthExtension"
    publisher            = "Microsoft.ManagedServices"
    type                 = "ApplicationHealthLinux"
    type_handler_version = "1.0"
    
    settings = jsonencode({
      protocol    = "http"
      port        = 80
      requestPath = "/health"
    })
  }
  
  tags = {
    environment = "prod"
    team        = "web"
  }
}

# Autoscale settings
resource "azurerm_monitor_autoscale_setting" "web" {
  name                = "as-vmss-web-prod"
  resource_group_name = azurerm_resource_group.web.name
  location            = azurerm_resource_group.web.location
  target_resource_id  = azurerm_orchestrated_virtual_machine_scale_set.web.id
  
  profile {
    name = "default"
    
    capacity {
      default = 3
      minimum = 2
      maximum = 10
    }
    
    # Scale-out rule (CPU > 70%)
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_orchestrated_virtual_machine_scale_set.web.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT10M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 70
      }
      
      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "2"
        cooldown  = "PT5M"
      }
    }
    
    # Scale-in rule (CPU < 30%)
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_orchestrated_virtual_machine_scale_set.web.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT10M"
        time_aggregation   = "Average"
        operator           = "LessThan"
        threshold          = 30
      }
      
      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT10M"
      }
    }
  }
  
  # Business hours profile
  profile {
    name = "business-hours"
    
    capacity {
      default = 6
      minimum = 6
      maximum = 15
    }
    
    recurrence {
      timezone = "India Standard Time"
      days     = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
      hours    = [8]
      minutes  = [0]
    }
    
    # Same scale rules as default...
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_orchestrated_virtual_machine_scale_set.web.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT10M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 70
      }
      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "2"
        cooldown  = "PT5M"
      }
    }
    
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_orchestrated_virtual_machine_scale_set.web.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT10M"
        time_aggregation   = "Average"
        operator           = "LessThan"
        threshold          = 30
      }
      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT10M"
      }
    }
  }
  
  notification {
    email {
      send_to_subscription_administrator    = true
      send_to_subscription_co_administrator = false
      custom_emails                         = ["devops@company.com"]
    }
  }
}

# Load Balancer for VMSS
resource "azurerm_lb" "web" {
  name                = "lb-web-prod"
  location            = azurerm_resource_group.web.location
  resource_group_name = azurerm_resource_group.web.name
  sku                 = "Standard"
  
  frontend_ip_configuration {
    name                 = "PublicIP"
    public_ip_address_id = azurerm_public_ip.lb.id
  }
}

resource "azurerm_lb_backend_address_pool" "web" {
  loadbalancer_id = azurerm_lb.web.id
  name            = "bp-web-servers"
}

resource "azurerm_lb_probe" "web" {
  loadbalancer_id = azurerm_lb.web.id
  name            = "probe-http-80"
  protocol        = "Http"
  port            = 80
  request_path    = "/health"
  interval_in_seconds = 10
}

resource "azurerm_lb_rule" "http" {
  loadbalancer_id                = azurerm_lb.web.id
  name                           = "rule-http"
  protocol                       = "Tcp"
  frontend_port                  = 80
  backend_port                   = 80
  frontend_ip_configuration_name = "PublicIP"
  backend_address_pool_ids       = [azurerm_lb_backend_address_pool.web.id]
  probe_id                       = azurerm_lb_probe.web.id
}
```

### Bicep

```bicep
resource vmss 'Microsoft.Compute/virtualMachineScaleSets@2024-03-01' = {
  name: 'vmss-web-prod'
  location: resourceGroup().location
  zones: ['1', '2', '3']
  sku: {
    name: 'Standard_D2s_v5'
    capacity: 3
  }
  properties: {
    orchestrationMode: 'Flexible'
    platformFaultDomainCount: 1
    virtualMachineProfile: {
      osProfile: {
        computerNamePrefix: 'web'
        adminUsername: 'azureuser'
        linuxConfiguration: {
          disablePasswordAuthentication: true
          ssh: {
            publicKeys: [{ path: '/home/azureuser/.ssh/authorized_keys', keyData: sshKey }]
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
          managedDisk: { storageAccountType: 'Premium_LRS' }
        }
      }
      networkProfile: {
        networkInterfaceConfigurations: [
          {
            name: 'nic-vmss'
            properties: {
              primary: true
              enableAcceleratedNetworking: true
              ipConfigurations: [
                {
                  name: 'internal'
                  properties: {
                    primary: true
                    subnet: { id: subnetId }
                    loadBalancerBackendAddressPools: [{ id: backendPoolId }]
                  }
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```

---

## Part 7: az CLI Reference

```bash
# Create VMSS (Flexible mode)
az vmss create \
  --resource-group rg-web-prod \
  --name vmss-web-prod \
  --orchestration-mode Flexible \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 3 \
  --zones 1 2 3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --lb lb-web-prod \
  --lb-sku Standard \
  --vnet-name vnet-prod \
  --subnet subnet-web \
  --storage-sku Premium_LRS \
  --upgrade-policy-mode Rolling \
  --max-batch-instance-percent 20 \
  --pause-time-between-batches PT60S \
  --custom-data cloud-init.yaml \
  --tags environment=prod team=web

# Scale manually
az vmss scale \
  --resource-group rg-web-prod \
  --name vmss-web-prod \
  --new-capacity 5

# Update model (change image)
az vmss update \
  --resource-group rg-web-prod \
  --name vmss-web-prod \
  --set virtualMachineProfile.storageProfile.imageReference.version=latest

# Trigger rolling upgrade
az vmss rolling-upgrade start \
  --resource-group rg-web-prod \
  --name vmss-web-prod

# Check rolling upgrade status
az vmss rolling-upgrade get-latest \
  --resource-group rg-web-prod \
  --name vmss-web-prod

# List instances
az vmss list-instances \
  --resource-group rg-web-prod \
  --name vmss-web-prod \
  --output table

# Reimage specific instance
az vmss reimage \
  --resource-group rg-web-prod \
  --name vmss-web-prod \
  --instance-id 3

# Set autoscale
az monitor autoscale create \
  --resource-group rg-web-prod \
  --resource vmss-web-prod \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name as-web-prod \
  --min-count 2 \
  --max-count 10 \
  --count 3

# Add scale-out rule
az monitor autoscale rule create \
  --resource-group rg-web-prod \
  --autoscale-name as-web-prod \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 2

# Add scale-in rule
az monitor autoscale rule create \
  --resource-group rg-web-prod \
  --autoscale-name as-web-prod \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1

# Enable auto-repair
az vmss update \
  --resource-group rg-web-prod \
  --name vmss-web-prod \
  --enable-automatic-repairs true \
  --automatic-repairs-grace-period PT30M \
  --automatic-repairs-action Replace

# Run command on all instances
az vmss run-command invoke \
  --resource-group rg-web-prod \
  --name vmss-web-prod \
  --instance-id "*" \
  --command-id RunShellScript \
  --scripts "apt-get update && apt-get upgrade -y"

# Delete VMSS
az vmss delete \
  --resource-group rg-web-prod \
  --name vmss-web-prod
```

---

## Part 8: Real-World Patterns

### Startup

```
Setup: VMSS behind Load Balancer

VMSS: vmss-web-prod
├── Size: B2s (burstable, cheap)
├── Image: Ubuntu 22.04 + custom cloud-init
├── Zones: 1, 2, 3 (across all AZs)
├── Instances: min 2, max 6, default 2
├── Autoscale: CPU > 70% → add 1, CPU < 30% → remove 1
├── Upgrade: Rolling, MaxSurge, 20% batch
├── Health: LB probe /health, auto-repair enabled
├── Behind: Standard Load Balancer (public)
└── No public IPs on VMs

Deploy:
├── Update custom image in Compute Gallery
├── Update VMSS model → rolling upgrade
└── Or: Custom Script Extension to pull latest app

Cost: 2× B2s = ~$70/month + LB $18/month = ~$90/month
```

### Mid-Size

```
Architecture: Multi-tier VMSS

Web tier: vmss-web-prod
├── D2s_v5, zones 1-2-3
├── Min 3, Max 15
├── Custom golden image (Packer-built)
├── Behind public Load Balancer
├── Auto-scale: CPU 70% + request count
├── Rolling upgrade, MaxSurge, 20% batch
├── Auto-repair: grace 30 min, Replace
└── Managed identity → Key Vault

App tier: vmss-app-prod
├── D4s_v5, zones 1-2-3
├── Min 3, Max 12
├── Behind internal Load Balancer
├── Auto-scale: CPU 60%
└── VNet peering with web tier

Workers: vmss-workers-spot
├── D4s_v5 Spot VMs (70-90% savings)
├── Min 0, Max 20
├── Auto-scale: Service Bus queue depth > 500
├── Eviction policy: Deallocate
├── Processing: Service Bus messages
└── Stateless, fault-tolerant

Deployment pipeline:
├── Packer builds golden image → Compute Gallery
├── Terraform updates VMSS model → rolling upgrade
├── Health probe validates new instances
├── Auto-rollback if unhealthy threshold exceeded
└── Notifications: Email + Teams webhook on scale events

Cost: ~$500-1500/month (with Spot savings)
```

### Enterprise

```
Multi-region, multi-tier VMSS platform:

Region 1 (Central India):
├── vmss-web-india (D4s_v5, 3 zones, 6-30 instances)
│   ├── Custom image from Compute Gallery (replicated)
│   ├── Behind Application Gateway (WAF + SSL)
│   ├── Auto-scale: CPU + request count + schedule
│   └── Rolling + MaxSurge + auto-repair
├── vmss-app-india (D8s_v5, 3 zones, 6-24 instances)
│   ├── Behind internal LB
│   └── Auto-scale: CPU 60%
└── vmss-workers-india (D4s_v5 Spot, 0-50 instances)
    └── Auto-scale: Service Bus queue depth

Region 2 (West Europe):
├── vmss-web-eu (same config, fewer instances)
├── vmss-app-eu
└── Traffic Manager: Performance routing between regions

Image management:
├── Packer CI pipeline → monthly golden images
├── Azure Compute Gallery: Versioned, cross-region replicated
├── CIS-hardened base images (security compliance)
├── Auto-update: Azure Update Manager (weekly patches)
└── Image version pinning in VMSS model (no "latest" in prod)

Upgrade strategy:
├── Canary: Update 1 instance manually → monitor 30 min
├── Rolling: 10% batch, 120s pause, MaxSurge
├── Cross-zone upgrade enabled
├── Auto-rollback: If > 20% unhealthy → pause
└── Blue-green (for major changes): New VMSS → swap LB

Security:
├── No public IPs (all behind App Gateway/LB)
├── Azure Bastion for SSH access
├── Managed identity → Key Vault (no passwords on VMs)
├── NSG: Locked down per tier
├── Disk encryption: Customer-managed keys
├── Microsoft Defender for Servers
├── Just-in-time VM access
├── OS Login via Azure AD (Entra ID)
└── Compliance: Azure Policy enforces standards

Monitoring:
├── Azure Monitor: CPU, memory, disk, network per VMSS
├── VM Insights: Performance + dependency mapping
├── Log Analytics: Syslog, app logs, security events
├── Alerts: Scale events, health probe failures, CPU > 90%
├── Dashboards: Per-VMSS Grafana dashboards
└── Cost alerts: Budget thresholds per VMSS

Cost optimization:
├── Spot VMs for workers (60-90% savings)
├── Reserved Instances for web/app tier (40-60% savings)
├── Savings Plans for flexible commitment
├── Auto-shutdown for dev/staging VMSS
├── Right-sizing: Review VM Insights recommendations
└── Cost: $5,000-20,000/month (with reservations + Spot)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Auto-managed fleet of identical VMs |
| Orchestration | Flexible (recommended) or Uniform (legacy) |
| Scaling | Manual, metric-based autoscale, schedule-based |
| Scale metrics | CPU, memory, network, disk, queue depth, custom |
| Upgrade: Manual | You trigger per-instance |
| Upgrade: Automatic | All at once (risky, for dev/test) |
| Upgrade: Rolling | Batch-by-batch, configurable pace |
| MaxSurge | Create new first, delete old after (zero-downtime) |
| Auto-repair | Replace/reimage/restart unhealthy instances |
| Health source | LB probe or Application Health Extension |
| Spot VMs | Up to 90% discount, auto-replace evicted |
| Zones | Spread across 1-3 availability zones |
| SLA | 99.99% (multi-zone) |
| AWS equivalent | Auto Scaling Group + Launch Template |
| GCP equivalent | Managed Instance Group |

---

## What's Next?

In the next chapter, we'll cover Azure Functions — serverless compute.

→ Next: [Chapter 17: Azure Functions](17-azure-functions.md)

---

*Last Updated: May 2026*
