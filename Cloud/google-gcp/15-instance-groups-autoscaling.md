# Chapter 15: GCP Instance Groups & Autoscaling

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Instance Templates](#part-1-instance-templates)
- [Part 2: Instance Groups (Concepts)](#part-2-instance-groups-concepts)
- [Part 3: Creating a Managed Instance Group (Full Walkthrough)](#part-3-creating-a-managed-instance-group-full-walkthrough)
- [Part 4: Autoscaling Policies (Deep Dive)](#part-4-autoscaling-policies-deep-dive)
- [Part 5: Autohealing](#part-5-autohealing)
- [Part 6: Rolling Updates & Canary Deployments](#part-6-rolling-updates--canary-deployments)
- [Part 7: Stateful MIGs](#part-7-stateful-migs)
- [Part 8: Terraform & gcloud CLI](#part-8-terraform--gcloud-cli)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Instance Groups are collections of VM instances that you manage as a single entity. Combined with autoscaling, they enable automatic scaling of your compute capacity based on demand. Managed Instance Groups (MIGs) are the foundation for scalable, highly available applications on GCP.

```
What you'll learn:
├── Instance Templates (blueprint for VMs)
│   ├── Full creation walkthrough
│   ├── Machine type, image, disk, networking
│   ├── Startup scripts & metadata
│   └── Regional vs global templates
├── Instance Groups
│   ├── Managed Instance Groups (MIG) — autoscaled, auto-healed
│   ├── Unmanaged Instance Groups (heterogeneous VMs)
│   ├── Zonal vs Regional MIGs
│   └── Stateful vs Stateless MIGs
├── MIG Creation (Full Walkthrough)
│   ├── Template selection
│   ├── Location (single zone vs multi-zone)
│   ├── Autoscaling configuration
│   ├── Autohealing (health checks)
│   └── Update policies (rolling, canary)
├── Autoscaling Policies
│   ├── CPU utilization
│   ├── Load balancing utilization
│   ├── Cloud Monitoring metrics (custom)
│   ├── Schedules (predictive)
│   └── Multiple signals combined
├── Autohealing
├── Rolling Updates & Canary Deployments
├── Stateful MIGs
├── Terraform & gcloud CLI
└── Real-world patterns
```

---

## Part 1: Instance Templates

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTANCE TEMPLATES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A blueprint/recipe for creating VM instances.                 │
│ Used by: Managed Instance Groups to create identical VMs.          │
│ Immutable: Once created, cannot be edited (create new version).   │
│                                                                       │
│ AWS equivalent: Launch Template                                     │
│ Azure equivalent: VM Scale Set model/image                         │
│                                                                       │
│ Types:                                                                │
│ ├── Global instance template (legacy, available everywhere)       │
│ └── Regional instance template (newer, recommended)               │
│     ⚡ Regional templates can reference regional resources        │
│        (regional disks, subnet in specific region)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Console → Compute Engine → Instance templates → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE INSTANCE TEMPLATE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [it-web-server-v1]                                            │
│ ⚡ Include version in name (immutable, so you create new versions) │
│   Naming: it-{purpose}-v{number} or it-{purpose}-{date}          │
│                                                                       │
│ Location:                                                            │
│   ● Global (available in all regions)                              │
│   ○ Regional: [asia-south1 ▼]                                     │
│                                                                       │
│ ── Machine configuration ──                                         │
│                                                                       │
│ Series: [E2 ▼] (same options as regular VM creation)              │
│ Machine type: [e2-medium ▼] (2 vCPU, 4 GB)                       │
│ ⚡ Or custom: vCPUs [4], Memory [8] GB                             │
│                                                                       │
│ ── Boot disk ──                                                     │
│                                                                       │
│ Image: [Ubuntu 22.04 LTS ▼]                                       │
│   ○ Public images (Debian, Ubuntu, CentOS, Windows, etc.)        │
│   ○ Custom images (your golden image with pre-installed software)│
│                                                                       │
│ Boot disk type: [Balanced persistent disk ▼]                      │
│ Size: [20] GB                                                       │
│ □ Delete boot disk when instance is deleted: ☑                    │
│                                                                       │
│ ── Networking ──                                                    │
│                                                                       │
│ Network: [vpc-prod ▼]                                               │
│ Subnet: [subnet-web-asia-south1 ▼]                                │
│ Network tags: [web-server, allow-health-check]                    │
│ ⚡ Tags used for firewall rules (allow LB health checks!)        │
│                                                                       │
│ External IP:                                                        │
│   ● Ephemeral (auto-assigned, changes on restart)                │
│   ○ None (no public IP — recommended for backend VMs)           │
│   ○ Static IP                                                     │
│                                                                       │
│ ── Identity and API access ──                                       │
│                                                                       │
│ Service account: [sa-web-server@project.iam.gserviceaccount.com ▼]│
│ Access scopes:                                                      │
│   ● Allow default access                                          │
│   ○ Allow full access to all Cloud APIs                          │
│   ○ Set access for each API                                      │
│   ⚡ Use dedicated service account (not default compute SA!)     │
│                                                                       │
│ ── Management ──                                                    │
│                                                                       │
│ Metadata:                                                            │
│ ┌──────────────────┬──────────────────────────────────────────┐   │
│ │ Key              │ Value                                      │   │
│ ├──────────────────┼──────────────────────────────────────────┤   │
│ │ startup-script   │ #!/bin/bash                               │   │
│ │                  │ apt-get update                             │   │
│ │                  │ apt-get install -y nginx                   │   │
│ │                  │ systemctl start nginx                      │   │
│ │ environment      │ prod                                      │   │
│ │ app-version      │ 2.1.0                                     │   │
│ └──────────────────┴──────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Or use startup-script-url to load from GCS:                      │
│   startup-script-url: gs://my-bucket/scripts/startup.sh           │
│                                                                       │
│ Availability policies:                                              │
│   On host maintenance: ● Migrate  ○ Terminate                    │
│   Automatic restart: ☑ On                                         │
│   Provisioning model: ● Standard  ○ Spot                         │
│                                                                       │
│ ── Security ──                                                      │
│                                                                       │
│ Shielded VM:                                                        │
│   ☑ Secure Boot                                                    │
│   ☑ vTPM                                                           │
│   ☑ Integrity Monitoring                                           │
│                                                                       │
│ ── Additional disks ──                                              │
│                                                                       │
│ [+ Add new disk]                                                    │
│ Name: [data-disk]                                                   │
│ Type: [SSD persistent disk]                                        │
│ Size: [100] GB                                                      │
│ Mode: Read/write                                                    │
│ Delete rule: ● Keep  ○ Delete (when instance deleted)            │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Instance Groups (Concepts)

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTANCE GROUPS: MANAGED vs UNMANAGED                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────────────────┬─────────────────────────────────────┐  │
│ │ Managed (MIG)           │ Unmanaged                           │  │
│ ├─────────────────────────┼─────────────────────────────────────┤  │
│ │ All VMs identical       │ Mix of different VMs                │  │
│ │ (from template)         │ (manually added)                    │  │
│ │ Autoscaling ✅          │ No autoscaling ❌                   │  │
│ │ Autohealing ✅          │ No autohealing ❌                   │  │
│ │ Rolling updates ✅      │ Manual updates ❌                   │  │
│ │ Multi-zone (regional) ✅│ Single zone only ❌                 │  │
│ │ Load balancer backend ✅│ Load balancer backend ✅            │  │
│ │ 99% of use cases       │ Legacy/special cases only          │  │
│ └─────────────────────────┴─────────────────────────────────────┘  │
│                                                                       │
│ ⚡ ALWAYS use Managed Instance Groups unless you have a very       │
│    specific reason not to (e.g., existing heterogeneous VMs).     │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ MIG types:                                                           │
│                                                                       │
│ 1. Zonal MIG:                                                       │
│    ├── All instances in ONE zone (e.g., asia-south1-a)           │
│    ├── If zone goes down → all instances down!                   │
│    └── Use for: Dev/test, single-zone workloads                  │
│                                                                       │
│ 2. Regional MIG (multi-zone):                                       │
│    ├── Instances spread across MULTIPLE zones in a region        │
│    ├── Automatic distribution (e.g., 3 zones × 2 VMs = 6 total)│
│    ├── If one zone fails → other zones continue                 │
│    ├── ⚡ ALWAYS use Regional MIG for production!                │
│    └── AWS equivalent: ASG with multiple AZs                    │
│                                                                       │
│ 3. Stateful MIG:                                                    │
│    ├── Preserves instance-specific state (disks, IPs, metadata) │
│    ├── VMs have identity (not disposable)                       │
│    ├── Use for: Databases, message queues, legacy apps         │
│    └── Autohealing recreates VM with SAME disk/IP               │
│                                                                       │
│ 4. Stateless MIG (default):                                        │
│    ├── VMs are disposable/interchangeable                       │
│    ├── No preserved state                                       │
│    └── Use for: Web servers, API servers, workers              │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Regional MIG (3 zones):                                      │   │
│ │                                                              │   │
│ │ ┌─────────────────┐ ┌─────────────────┐ ┌───────────────┐ │   │
│ │ │ asia-south1-a   │ │ asia-south1-b   │ │ asia-south1-c │ │   │
│ │ │ ┌──┐ ┌──┐      │ │ ┌──┐ ┌──┐      │ │ ┌──┐ ┌──┐    │ │   │
│ │ │ │VM│ │VM│      │ │ │VM│ │VM│      │ │ │VM│ │VM│    │ │   │
│ │ │ └──┘ └──┘      │ │ └──┘ └──┘      │ │ └──┘ └──┘    │ │   │
│ │ └─────────────────┘ └─────────────────┘ └───────────────┘ │   │
│ │              ↑                                               │   │
│ │   All created from same Instance Template                   │   │
│ │   Total: 6 VMs, distributed evenly across 3 zones          │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating a Managed Instance Group (Full Walkthrough)

```
Console → Compute Engine → Instance groups → Create instance group

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE MANAGED INSTANCE GROUP                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── New managed instance group (stateless) ──                        │
│   ○ New managed instance group (stateful)                          │
│   ○ New unmanaged instance group                                   │
│                                                                       │
│ Name: [mig-web-servers]                                             │
│ Description: [Production web server fleet]                          │
│                                                                       │
│ ── Instance template ──                                             │
│                                                                       │
│ Instance template: [it-web-server-v1 ▼]                            │
│ ⚡ The template defines what each VM looks like.                    │
│   Change template → rolling update to deploy new version.        │
│                                                                       │
│ ── Location ──                                                      │
│                                                                       │
│   ○ Single zone                                                    │
│   ● Multiple zones (recommended for production)                   │
│                                                                       │
│ Region: [asia-south1 ▼]                                            │
│ Zones: ☑ asia-south1-a  ☑ asia-south1-b  ☑ asia-south1-c         │
│ ⚡ Select all available zones for maximum availability.            │
│                                                                       │
│ Target distribution shape:                                          │
│   ● Even (default — equal VMs per zone)                           │
│   ○ Balanced (best-effort even, respects capacity)               │
│   ○ Any (no zone preference — fastest scaling)                   │
│   ○ Any single zone (all in one zone, fails over to another)    │
│                                                                       │
│ ── Autoscaling ──                                                   │
│                                                                       │
│ Autoscaling mode:                                                    │
│   ● Autoscale (scale in AND out automatically)                    │
│   ○ Only scale out (never removes instances)                     │
│   ○ Off (fixed size — no autoscaling)                            │
│                                                                       │
│ Minimum number of instances: [3]                                   │
│ ⚡ At least 1 per zone for regional MIG (3 zones → min 3)        │
│                                                                       │
│ Maximum number of instances: [12]                                  │
│ ⚡ Cap to control costs. Size according to max expected load.     │
│                                                                       │
│ ── Autoscaling signals ──                                           │
│                                                                       │
│ [+ Add a signal]                                                    │
│                                                                       │
│ Signal type: [CPU utilization ▼]                                  │
│   ● CPU utilization                                                │
│   ○ HTTP load balancing utilization                               │
│   ○ Cloud Monitoring metric                                       │
│   ○ Schedule                                                       │
│                                                                       │
│ ── CPU utilization ──                                               │
│ Target CPU utilization: [60] %                                     │
│ ⚡ When avg CPU > 60% → add instances                             │
│ ⚡ When avg CPU < 60% → remove instances (cool down first)       │
│ ⚡ 60% recommended (headroom for spikes)                          │
│                                                                       │
│ Predictive autoscaling: [Off ▼]                                   │
│   ○ Off                                                            │
│   ○ Optimize for availability (scale up early based on forecast)│
│   ○ Only forecast (don't act, just predict)                      │
│   ⚡ Uses ML to predict traffic patterns (like business hours)   │
│                                                                       │
│ ── Scale-in controls ──                                             │
│                                                                       │
│ □ Enable scale-in controls                                         │
│ Don't scale in by more than: [10] % of instances                 │
│ Over: [10] minutes                                                  │
│ ⚡ Prevents aggressive scale-down during brief traffic dips.     │
│                                                                       │
│ Cool down period: [60] seconds                                     │
│ ⚡ Wait time after scaling before evaluating again.               │
│                                                                       │
│ ── Initialization period ──                                         │
│                                                                       │
│ Initialization period: [300] seconds                               │
│ ⚡ Time for new VM to boot + startup script to complete.          │
│   Autoscaler ignores new VMs during this period.                 │
│   Set to actual boot time (including app startup).               │
│                                                                       │
│ ── Autohealing ──                                                   │
│                                                                       │
│ Health check: [hc-web-http ▼]                                     │
│ ⚡ If VM fails health check → automatically recreated!           │
│                                                                       │
│ Initial delay: [300] seconds                                       │
│ ⚡ Grace period after VM creation before health checks start.    │
│   Must be >= time for VM to boot and become healthy.            │
│   Too short → autohealer kills VMs still starting up!           │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Autoscaling Policies (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTOSCALING SIGNALS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. CPU Utilization:                                                  │
│    ├── Target: 60% (most common)                                 │
│    ├── If average CPU across group > target → scale out          │
│    ├── If average CPU across group < target → scale in           │
│    ├── Mode: ● Target utilization  ○ Single instance assignment │
│    └── Simple, works for CPU-bound workloads                     │
│                                                                       │
│ 2. HTTP Load Balancing Utilization:                                 │
│    ├── Target: requests per instance (e.g., 100 RPS/instance)   │
│    ├── Autoscaler gets data from the HTTP(S) load balancer       │
│    ├── Better than CPU for web workloads with I/O wait          │
│    └── Requires: MIG must be backend of HTTP(S) LB              │
│                                                                       │
│ 3. Cloud Monitoring Metric (custom):                                │
│    ├── Any metric from Cloud Monitoring                          │
│    ├── Examples:                                                   │
│    │   ├── Queue depth (Pub/Sub messages undelivered)           │
│    │   ├── Custom application metric (active connections)       │
│    │   └── Memory utilization (via Ops Agent)                   │
│    ├── Utilization target: per-instance metric (e.g., 80%)      │
│    ├── Single instance assignment: metric value per VM           │
│    └── ⚡ Most flexible — scale on ANY business metric           │
│                                                                       │
│    Filter: resource.type = "gce_instance"                        │
│    Metric: custom.googleapis.com/my_app/active_connections      │
│    Target: 500 connections per instance                          │
│                                                                       │
│ 4. Schedule-based Autoscaling:                                      │
│    ├── Set minimum instances on a schedule                       │
│    ├── Cron-like: "Every weekday 8 AM → min 10 instances"       │
│    ├── "Every weekend → min 3 instances"                        │
│    ├── Multiple schedules can overlap (highest min wins)        │
│    └── ⚡ Use for predictable traffic patterns                    │
│                                                                       │
│    Schedule config:                                                 │
│    Name: [business-hours]                                          │
│    Min instances: [10]                                              │
│    Start time: 2026-01-01T08:00:00Z                               │
│    Duration: [12h] (or End time)                                   │
│    Recurrence: [FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR]               │
│    Timezone: [Asia/Kolkata]                                        │
│                                                                       │
│ ── MULTIPLE SIGNALS ──                                              │
│                                                                       │
│ You can configure MULTIPLE signals simultaneously!                 │
│ Autoscaler uses the signal that recommends the MOST instances.   │
│                                                                       │
│ Example:                                                             │
│ ├── Signal 1: CPU > 60% → recommends 8 instances               │
│ ├── Signal 2: Schedule → requires minimum 6 instances           │
│ ├── Signal 3: LB utilization → recommends 10 instances          │
│ └── Result: Autoscaler uses 10 (the highest)                    │
│                                                                       │
│ ── SCALING BEHAVIOR ──                                              │
│                                                                       │
│ Scale out:                                                           │
│ ├── Fast — adds instances quickly to handle load                │
│ ├── Multiple instances can be added at once                     │
│ └── New instances go through initialization period              │
│                                                                       │
│ Scale in:                                                            │
│ ├── Gradual — conservative to avoid thrashing                   │
│ ├── Cool-down period between scale-in actions                   │
│ ├── Scale-in controls can limit how fast (% per time)          │
│ └── Prefers removing newest instances (or from zone with most) │
│                                                                       │
│ Scale-in controls (recommended for production):                    │
│ ├── Don't scale in by more than: 10%                            │
│ ├── Over: 10 minutes                                             │
│ ├── Prevents: One bad metric reading killing instances          │
│ └── Prevents: Brief dips in traffic causing premature scale-in │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Autohealing

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTOHEALING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Automatically detect and replace unhealthy VM instances.     │
│                                                                       │
│ Flow:                                                                │
│ Health check probe → VM unresponsive → Mark unhealthy            │
│ → Delete unhealthy VM → Create new VM from template              │
│                                                                       │
│ ⚡ This is NOT the load balancer health check (though similar).    │
│    MIG has its OWN health check for autohealing.                 │
│    You can use the same health check resource for both.          │
│                                                                       │
│ Health check creation:                                              │
│ Console → Compute Engine → Health checks → Create                │
│                                                                       │
│ Name: [hc-web-http]                                                 │
│ Protocol: [HTTP ▼]  (HTTP, HTTPS, TCP, SSL, HTTP2)                │
│ Port: [80]                                                          │
│ Request path: [/health]                                             │
│                                                                       │
│ Health criteria:                                                     │
│ ├── Check interval: [10] seconds (how often to probe)            │
│ ├── Timeout: [5] seconds (wait for response)                    │
│ ├── Healthy threshold: [2] consecutive successes                │
│ └── Unhealthy threshold: [3] consecutive failures               │
│                                                                       │
│ ⚡ Initial delay (set on MIG, not health check):                   │
│   Time to wait before starting health checks on a new VM.       │
│   CRITICAL: Set this to >= your VM boot + app startup time!     │
│   Too short (60s) → new VMs killed before they're ready!        │
│   Recommended: 300s (5 min) for typical web apps.               │
│                                                                       │
│ What happens when VM is unhealthy:                                  │
│ 1. Health check fails (3 consecutive times at 10s interval = 30s)│
│ 2. MIG marks instance as UNHEALTHY                                │
│ 3. MIG deletes the unhealthy instance                            │
│ 4. MIG creates a new instance from the template                  │
│ 5. New instance goes through initial delay before health checks  │
│                                                                       │
│ ⚡ Stateful MIG: Recreates with same disk/IP (preserves state)    │
│ ⚡ Stateless MIG: Brand new VM (no state preserved)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Rolling Updates & Canary Deployments

```
┌─────────────────────────────────────────────────────────────────────┐
│           UPDATE POLICIES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When you change the instance template on a MIG, existing VMs      │
│ need to be updated. Update policy controls HOW this happens.      │
│                                                                       │
│ Console → MIG → Update VMs                                        │
│                                                                       │
│ Update type:                                                        │
│   ● Proactive (automatic — start updating immediately)           │
│   ○ Opportunistic (only update when VMs are recreated)          │
│   ⚡ Proactive = rolling update. Opportunistic = lazy update.    │
│                                                                       │
│ ── Rolling Update Configuration ──                                  │
│                                                                       │
│ Maximum surge: [3] instances (or 20%)                             │
│ ⚡ How many EXTRA instances to create during update.              │
│   More surge = faster update (but more cost during update)       │
│   0 surge = replacement style (delete old first, then create new)│
│                                                                       │
│ Maximum unavailable: [0] instances (or 0%)                       │
│ ⚡ How many instances can be unavailable during update.           │
│   0 = zero-downtime update (always maintain full capacity)      │
│   Higher = faster update but reduced capacity                   │
│                                                                       │
│ ⚡ Recommended for production (zero-downtime):                     │
│   Surge: 3 (or 25%)                                              │
│   Unavailable: 0                                                  │
│   → Creates 3 new instances → waits until healthy →             │
│     removes 3 old instances → repeat until done                 │
│                                                                       │
│ Minimum wait time: [0] seconds                                     │
│ ⚡ Wait between updating batches. Set 60-120s for monitoring.    │
│                                                                       │
│ ── CANARY DEPLOYMENT ──                                             │
│                                                                       │
│ Canary = deploy new template to a SUBSET of instances.           │
│                                                                       │
│ Method:                                                              │
│ 1. Create new instance template (it-web-server-v2)               │
│ 2. MIG → Rolling update → Set target size                       │
│    Template 1 (it-web-server-v1): [90]%                          │
│    Template 2 (it-web-server-v2): [10]%                          │
│ 3. Monitor canary VMs (errors, latency, etc.)                   │
│ 4. If good → change to 0% v1, 100% v2                          │
│ 5. If bad → rollback: 100% v1, 0% v2                           │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ MIG: mig-web-servers (12 instances)                          │  │
│ │                                                              │  │
│ │ Template v1 (90%):  [VM][VM][VM][VM][VM][VM][VM][VM][VM][VM]│  │
│ │ Template v2 (10%):  [VM][VM] ← canary (new version)        │  │
│ │                                                              │  │
│ │ Monitor → if good → shift to 100% v2                       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ gcloud command for canary:                                          │
│ gcloud compute instance-groups managed rolling-action             │
│   start-update mig-web-servers \                                  │
│   --version=template=it-web-server-v1 \                          │
│   --canary-version=template=it-web-server-v2,target-size=10% \  │
│   --region=asia-south1                                            │
│                                                                       │
│ Full rollout:                                                       │
│ gcloud compute instance-groups managed rolling-action             │
│   start-update mig-web-servers \                                  │
│   --version=template=it-web-server-v2 \                          │
│   --region=asia-south1                                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Stateful MIGs

```
┌─────────────────────────────────────────────────────────────────────┐
│           STATEFUL MANAGED INSTANCE GROUPS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: MIG instances that preserve state across recreation.         │
│                                                                       │
│ Stateful config (per instance):                                     │
│ ├── Preserved disks (data disk survives recreation)              │
│ ├── Preserved internal IPs                                       │
│ ├── Preserved external IPs                                       │
│ └── Preserved metadata (key-value pairs)                         │
│                                                                       │
│ Use cases:                                                           │
│ ├── Databases (need persistent disk)                            │
│ ├── Message queues (Kafka, RabbitMQ)                            │
│ ├── Legacy apps with local state                                │
│ ├── Batch processing with checkpoints                           │
│ └── Gaming servers (persistent world state)                     │
│                                                                       │
│ Stateful policy (applied to ALL instances in MIG):                 │
│ Console → MIG → Details → Stateful configuration                 │
│                                                                       │
│ Preserved state:                                                    │
│ ├── Disk: data-disk → On permanent deletion: ● Never delete     │
│ │                                            ○ Delete           │
│ ├── Internal IP: nic0 → Preserve on instance update            │
│ └── External IP: nic0 → Preserve on instance update            │
│                                                                       │
│ Per-instance config (for specific instances):                      │
│ gcloud compute instance-groups managed instance-configs create \  │
│   mig-web-servers \                                                │
│   --instance=instance-abc123 \                                     │
│   --stateful-disk=device-name=data-disk,\                         │
│     source=projects/proj/zones/zone/disks/my-data-disk \          │
│   --region=asia-south1                                             │
│                                                                       │
│ ⚡ When autohealing recreates a stateful instance:                  │
│   ├── Same persistent disk is reattached                        │
│   ├── Same internal IP is assigned                              │
│   ├── Same metadata preserved                                   │
│   └── Like "replacing the server but keeping the hard drive"   │
│                                                                       │
│ ⚠️ Limitations of stateful MIGs:                                    │
│ ├── Autoscaling works but scale-in may delete stateful instances│
│ ├── Rolling updates more complex (must handle state migration) │
│ └── Consider: Is a managed database (Cloud SQL) better?        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & gcloud CLI

### Terraform

```hcl
# Instance template
resource "google_compute_instance_template" "web" {
  name         = "it-web-server-v1"
  machine_type = "e2-medium"
  region       = "asia-south1"
  
  tags = ["web-server", "allow-health-check"]
  
  disk {
    source_image = "debian-cloud/debian-12"
    auto_delete  = true
    boot         = true
    disk_size_gb = 20
    disk_type    = "pd-balanced"
  }
  
  network_interface {
    network    = google_compute_network.vpc.id
    subnetwork = google_compute_subnetwork.web.id
    # No access_config = no external IP (recommended)
  }
  
  service_account {
    email  = google_service_account.web.email
    scopes = ["cloud-platform"]
  }
  
  metadata = {
    startup-script = <<-EOF
      #!/bin/bash
      apt-get update
      apt-get install -y nginx
      systemctl start nginx
    EOF
  }
  
  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }
  
  scheduling {
    automatic_restart   = true
    on_host_maintenance = "MIGRATE"
    preemptible         = false
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# Health check (for autohealing + LB)
resource "google_compute_health_check" "web" {
  name                = "hc-web-http"
  check_interval_sec  = 10
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3
  
  http_health_check {
    port         = 80
    request_path = "/health"
  }
}

# Regional Managed Instance Group
resource "google_compute_region_instance_group_manager" "web" {
  name               = "mig-web-servers"
  region             = "asia-south1"
  base_instance_name = "web"
  
  version {
    instance_template = google_compute_instance_template.web.id
    name              = "primary"
  }
  
  # For canary deployment, add second version:
  # version {
  #   instance_template = google_compute_instance_template.web_v2.id
  #   name              = "canary"
  #   target_size {
  #     percent = 10
  #   }
  # }
  
  target_size = 3  # Remove if using autoscaler
  
  named_port {
    name = "http"
    port = 80
  }
  
  auto_healing_policies {
    health_check      = google_compute_health_check.web.id
    initial_delay_sec = 300
  }
  
  update_policy {
    type                           = "PROACTIVE"
    minimal_action                 = "REPLACE"
    most_disruptive_allowed_action = "REPLACE"
    max_surge_fixed                = 3
    max_unavailable_fixed          = 0
    replacement_method             = "SUBSTITUTE"
  }
  
  distribution_policy_zones = [
    "asia-south1-a",
    "asia-south1-b",
    "asia-south1-c",
  ]
  
  distribution_policy_target_shape = "EVEN"
}

# Autoscaler
resource "google_compute_region_autoscaler" "web" {
  name   = "as-web-servers"
  region = "asia-south1"
  target = google_compute_region_instance_group_manager.web.id
  
  autoscaling_policy {
    min_replicas    = 3
    max_replicas    = 12
    cooldown_period = 60
    
    cpu_utilization {
      target = 0.6  # 60%
    }
    
    # Optional: Scale-in controls
    scale_in_control {
      max_scaled_in_replicas {
        percent = 10
      }
      time_window_sec = 600  # 10 minutes
    }
  }
}

# Schedule-based autoscaling
resource "google_compute_region_autoscaler" "web_scheduled" {
  name   = "as-web-scheduled"
  region = "asia-south1"
  target = google_compute_region_instance_group_manager.web.id
  
  autoscaling_policy {
    min_replicas    = 3
    max_replicas    = 20
    cooldown_period = 60
    
    cpu_utilization {
      target = 0.6
    }
    
    scaling_schedules {
      name                  = "business-hours"
      min_required_replicas = 10
      schedule              = "0 8 * * MON-FRI"
      time_zone             = "Asia/Kolkata"
      duration_sec          = 43200  # 12 hours
    }
    
    scaling_schedules {
      name                  = "weekend"
      min_required_replicas = 3
      schedule              = "0 0 * * SAT"
      time_zone             = "Asia/Kolkata"
      duration_sec          = 172800  # 48 hours
    }
  }
}
```

### gcloud CLI

```bash
# Create instance template
gcloud compute instance-templates create it-web-server-v1 \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-balanced \
  --network=vpc-prod \
  --subnet=subnet-web-asia-south1 \
  --region=asia-south1 \
  --tags=web-server,allow-health-check \
  --service-account=sa-web@project-id.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --metadata-from-file=startup-script=startup.sh \
  --shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring \
  --no-address  # No external IP

# Create health check
gcloud compute health-checks create http hc-web-http \
  --port=80 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3

# Create regional MIG
gcloud compute instance-groups managed create mig-web-servers \
  --template=it-web-server-v1 \
  --size=3 \
  --region=asia-south1 \
  --zones=asia-south1-a,asia-south1-b,asia-south1-c \
  --health-check=hc-web-http \
  --initial-delay=300

# Set named port (for load balancer)
gcloud compute instance-groups set-named-ports mig-web-servers \
  --named-ports=http:80 \
  --region=asia-south1

# Set autoscaling
gcloud compute instance-groups managed set-autoscaling mig-web-servers \
  --region=asia-south1 \
  --min-num-replicas=3 \
  --max-num-replicas=12 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=60 \
  --scale-in-control=max-scaled-in-replicas=10%,time-window=600

# Add scheduled scaling
gcloud compute instance-groups managed update-autoscaling mig-web-servers \
  --region=asia-south1 \
  --set-schedule=business-hours \
  --schedule-cron="0 8 * * MON-FRI" \
  --schedule-min-required-replicas=10 \
  --schedule-duration-sec=43200 \
  --schedule-time-zone="Asia/Kolkata"

# Rolling update (deploy new template)
gcloud compute instance-groups managed rolling-action start-update mig-web-servers \
  --version=template=it-web-server-v2 \
  --max-surge=3 \
  --max-unavailable=0 \
  --region=asia-south1

# Canary deployment (10% new version)
gcloud compute instance-groups managed rolling-action start-update mig-web-servers \
  --version=template=it-web-server-v1 \
  --canary-version=template=it-web-server-v2,target-size=10% \
  --region=asia-south1

# Rollback (back to v1)
gcloud compute instance-groups managed rolling-action start-update mig-web-servers \
  --version=template=it-web-server-v1 \
  --region=asia-south1

# Resize manually
gcloud compute instance-groups managed resize mig-web-servers \
  --size=6 \
  --region=asia-south1

# List instances in MIG
gcloud compute instance-groups managed list-instances mig-web-servers \
  --region=asia-south1

# Check autoscaler status
gcloud compute instance-groups managed describe mig-web-servers \
  --region=asia-south1 \
  --format="table(status.autoscaler.status, status.autoscaler.recommendedSize)"

# Delete MIG
gcloud compute instance-groups managed delete mig-web-servers \
  --region=asia-south1
```

---

## Part 9: Real-World Patterns

### Startup

```
Setup: Simple web tier with autoscaling

Instance template: it-web-v1
├── e2-small (2 vCPU, 2 GB) — cheap but functional
├── Boot: Debian 12, 10 GB pd-balanced
├── Startup script: install app, start service
├── No external IP (behind LB)
└── Tags: web-server, allow-health-check

MIG: mig-web
├── Regional (asia-south1, 2-3 zones)
├── Min: 2, Max: 6
├── Autoscale: CPU 70%
├── Autohealing: HTTP /health, initial delay 300s
└── Update: Proactive, surge 1, unavailable 0

Deploy flow:
1. Build new image (or update startup script)
2. Create new template it-web-v2
3. Update MIG → rolling update (zero-downtime)

Cost: 2× e2-small = ~$25/month + auto-scales as needed
```

### Mid-Size

```
Architecture: Multi-tier with separate MIGs

Web tier: mig-web
├── Template: e2-medium, custom image (pre-baked)
├── Regional (3 zones), Min 3, Max 15
├── Autoscale: CPU 60% + LB utilization
├── Predictive autoscaling: ON (learns traffic patterns)
├── Scale-in controls: max 10% per 10 min
└── Behind: External HTTP(S) Load Balancer

App tier: mig-app
├── Template: n2-standard-4, custom image
├── Regional (3 zones), Min 3, Max 12
├── Autoscale: Custom metric (queue depth from Pub/Sub)
└── Behind: Internal TCP Load Balancer

Worker tier: mig-workers
├── Template: c2-standard-8 (compute-optimized)
├── SPOT VMs (80% cheaper!)
├── Min 0, Max 20
├── Autoscale: Custom metric (Pub/Sub unacked messages)
└── Preemption handling: graceful shutdown script

Deployment:
├── Custom images built in CI (Packer)
├── New image → new template → canary 10% → full rollout
├── Rollback: point MIG back to previous template
└── Blue-green: Create new MIG, shift LB traffic, delete old

Monitoring:
├── Cloud Monitoring: CPU, memory, disk, custom app metrics
├── Autoscaler dashboard (recommended size vs actual)
├── Alerts: Scaling events, health check failures
└── Uptime checks (external probing)

Cost: ~$500-2000/month (depending on scaling)
```

### Enterprise

```
Multi-region, multi-tier with advanced strategies:

Global architecture:
├── Region 1 (asia-south1): Primary
│   ├── mig-web-asia (Regional, 3 zones, 6-30 instances)
│   ├── mig-app-asia (Regional, 3 zones, 6-24 instances)
│   └── mig-workers-asia (Spot, 0-50 instances)
├── Region 2 (us-central1): Secondary
│   ├── mig-web-us (Regional, 3 zones, 3-20 instances)
│   ├── mig-app-us (Regional, 3 zones, 3-16 instances)
│   └── mig-workers-us (Spot, 0-30 instances)
└── Global HTTP(S) LB distributes traffic

Autoscaling strategy:
├── CPU (60%) + LB utilization + custom metrics
├── Predictive autoscaling enabled
├── Scheduled scaling:
│   ├── Business hours India (8AM-8PM IST): min 10 web
│   ├── Business hours US (8AM-8PM EST): min 8 web
│   └── Off-hours: min 3 per region
├── Scale-in controls: 10% per 10 minutes
└── Initialization period: 300s (custom image boots fast)

Instance templates:
├── Golden images built weekly (Packer + CI)
├── Security patches baked into images
├── Minimal startup script (image has everything)
├── ARM instances where possible (T2A — cost savings)
└── Confidential VMs for sensitive workloads

Update strategy:
├── Canary: 5% new version (per region)
├── Monitor 30 min: error rate, latency P99, logs
├── Auto-promote: 25% → 50% → 100%
├── Auto-rollback: if error rate > threshold
└── Regional rollout: Asia first → US after 1 hour

Spot VMs (workers):
├── 60-80% cost savings for batch/worker workloads
├── Preemption handler: flush work, checkpoint state
├── Mix regular + Spot for baseline + burst
└── Instance group has mixed regular (base) + Spot (burst)

Security:
├── No external IPs on ANY instance (all behind LB)
├── Dedicated service accounts per MIG (least privilege)
├── Shielded VMs mandatory
├── OS Login enabled (no SSH keys in metadata)
├── Custom images hardened (CIS benchmarks)
└── VPC Service Controls for data exfiltration prevention

Cost: $5,000-20,000/month (with Spot savings)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| Instance Template | Immutable blueprint for VMs in a MIG |
| MIG (Managed) | Identical VMs, autoscaling, autohealing, rolling updates |
| Unmanaged IG | Manual, heterogeneous VMs (legacy only) |
| Zonal MIG | All instances in one zone (dev/test) |
| Regional MIG | Multi-zone distribution (production) |
| Stateful MIG | Preserves disks/IPs across recreation |
| Autoscale signals | CPU, LB utilization, custom metrics, schedules |
| Scale-in controls | Limit scale-down speed (prevent thrashing) |
| Predictive scaling | ML-based forecast (pre-warm for expected traffic) |
| Autohealing | Replace unhealthy VMs automatically |
| Rolling update | Zero-downtime update with surge/unavailable config |
| Canary | Deploy new template to subset (10%) then expand |
| AWS equivalent | Auto Scaling Group + Launch Template |
| Azure equivalent | VM Scale Sets |

---

## Console Walkthrough: Managing & Deleting MIGs

### Editing Autoscaling Settings on an Existing MIG

```
Console → Compute Engine → Instance groups → [click your MIG]
→ Edit

┌─────────────────────────────────────────────────────────────────────┐
│           EDIT AUTOSCALING ON EXISTING MIG                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scroll to "Autoscaling" section:                                   │
│                                                                       │
│ Autoscaling mode:                                                    │
│   ● Autoscale (scale in AND out)                                    │
│   ○ Only scale out                                                  │
│   ○ Off (fixed number of instances)                                │
│                                                                       │
│ Minimum number of instances: [3] → change to [5]                  │
│ Maximum number of instances: [12] → change to [20]                │
│                                                                       │
│ Autoscaling signals:                                                │
│   CPU utilization: [60]% → change to [70]%                        │
│   ⚡ Or add new signals (Schedule, Cloud Monitoring metric, etc.)  │
│                                                                       │
│ Cool down period: [60]s → change to [120]s                        │
│                                                                       │
│ Scale-in controls:                                                  │
│   ☑ Enable → Don't scale in by more than [10]%                   │
│                                                                       │
│ [Save]                                                               │
│                                                                       │
│ ⚡ Changes take effect immediately — no MIG recreation needed.     │
│ ⚡ If you lower the max and current count exceeds it, MIG will    │
│   gradually scale in to meet the new max.                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Changing Instance Template on an Existing MIG

```
Console → Compute Engine → Instance groups → [click your MIG]
→ Edit

┌─────────────────────────────────────────────────────────────────────┐
│           CHANGE TEMPLATE ON EXISTING MIG                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Instance template: [it-web-server-v1 ▼]                            │
│   → Change to:     [it-web-server-v2 ▼]                            │
│                                                                       │
│ ⚡ Changing the template does NOT immediately replace existing VMs! │
│   It only affects NEW instances created after this change.         │
│                                                                       │
│ To update EXISTING instances to the new template:                  │
│ → Click "Update VMs" (at the top of the MIG page)                 │
│                                                                       │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │ UPDATE VMs                                                     │  │
│ │                                                                │  │
│ │ New template: [it-web-server-v2]                              │  │
│ │                                                                │  │
│ │ Update type:                                                   │  │
│ │   ○ Automatic (rolling update — GCP handles it)              │  │
│ │   ○ Selective (you pick which instances to update)           │  │
│ │                                                                │  │
│ │ (For Automatic — Rolling update config)                       │  │
│ │ Max surge: [3] instances (extra VMs during update)           │  │
│ │ Max unavailable: [0] instances (zero-downtime)               │  │
│ │                                                                │  │
│ │ Replacement method:                                           │  │
│ │   ● Substitute (delete old, create new)                      │  │
│ │   ○ Recreate (in-place restart)                              │  │
│ │                                                                │  │
│ │ [Update VMs]                                                   │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ You must create the new template BEFORE changing it on the MIG. │
│   Templates are immutable — you can't edit them, only create new. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Deleting a Managed Instance Group

```
Console → Compute Engine → Instance groups → [select MIG checkbox]
→ Delete (top bar)

┌─────────────────────────────────────────────────────────────────────┐
│           DELETE MIG                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️  "Are you sure you want to delete 'mig-web-servers'?"            │
│                                                                       │
│ What happens when you delete a MIG:                                 │
│ ├── ALL VM instances in the group are DELETED                     │
│ ├── Boot disks are deleted (if "delete boot disk" was checked    │
│ │   in the template — which is the default)                      │
│ ├── Data disks: depends on deletion rule set in template          │
│ │   ├── "Delete" → data disk also deleted                       │
│ │   └── "Keep" → data disk is preserved (detached)              │
│ ├── The instance template is NOT deleted (still reusable)        │
│ ├── Health checks are NOT deleted (still reusable)               │
│ └── Load balancer backend reference becomes invalid               │
│     (you'll need to remove it from the LB backend service)      │
│                                                                       │
│ ⚡ Deleting is permanent — all VMs are immediately terminated.     │
│ ⚡ Take snapshots of important disks BEFORE deleting the MIG.      │
│ ⚡ If the MIG is behind a load balancer, remove it from the        │
│   backend service first to avoid errors.                           │
│                                                                       │
│ [Delete]   [Cancel]                                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Deleting an Instance Template

```
Console → Compute Engine → Instance templates → [select template checkbox]
→ Delete (top bar)

┌─────────────────────────────────────────────────────────────────────┐
│           DELETE INSTANCE TEMPLATE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️  "Are you sure you want to delete 'it-web-server-v1'?"           │
│                                                                       │
│ Prerequisites:                                                      │
│ ├── Template must NOT be in use by any MIG                        │
│ │   ⚠️ If a MIG is using this template, you CANNOT delete it!     │
│ │   First change the MIG to use a different template.            │
│ └── No other resources referencing this template                  │
│                                                                       │
│ What happens:                                                       │
│ ├── Template is permanently deleted                               │
│ ├── Existing VMs that were created from this template are         │
│ │   NOT affected (they continue running)                          │
│ └── You cannot undo this — create a new template if needed       │
│                                                                       │
│ ⚡ Keep old templates around until you're sure the new version     │
│   is stable — easy rollback by changing MIG back to old template. │
│                                                                       │
│ [Delete]   [Cancel]                                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover GCP Cloud Functions — the serverless compute service.

→ Next: [Chapter 16: Cloud Functions](16-cloud-functions.md)

---

*Last Updated: May 2026*
