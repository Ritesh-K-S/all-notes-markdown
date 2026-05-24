# Chapter 41: Azure Service Health & Advisor

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Service Health](#part-1-azure-service-health)
- [Part 2: Configuring Health Alerts (Portal Walkthrough)](#part-2-configuring-health-alerts-portal-walkthrough)
- [Part 3: Azure Advisor](#part-3-azure-advisor)
- [Part 4: Advisor Recommendations (Portal Walkthrough)](#part-4-advisor-recommendations-portal-walkthrough)
- [Part 5: Azure Resource Health](#part-5-azure-resource-health)
- [Part 6: az CLI Reference](#part-6-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Service Health tells you when Azure itself has problems (outages, maintenance, advisories). Azure Advisor gives you personalized recommendations to improve your Azure setup — like unused resources you're paying for, security issues, or performance improvements.

```
What you'll learn:
├── Azure Service Health
│   ├── Service issues (Azure outages)
│   ├── Planned maintenance (upcoming changes)
│   ├── Health advisories (feature retirements)
│   └── Setting up health alerts
├── Azure Advisor
│   ├── Cost recommendations (save money)
│   ├── Security recommendations
│   ├── Reliability recommendations
│   ├── Performance recommendations
│   └── Operational excellence
├── Azure Resource Health (per-resource health)
└── az CLI reference
```

---

## Part 1: Azure Service Health

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE SERVICE HEALTH                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Service Health                                           │
│                                                                       │
│ Three categories:                                                    │
│                                                                       │
│ 1. Service issues (outages) 🔴                                    │
│    ├── Azure services experiencing problems                      │
│    ├── Only shows issues affecting YOUR resources                │
│    ├── Example: "App Service in Central India experiencing errors"│
│    ├── Timeline, affected services, updates, root cause analysis │
│    └── Post-incident: Root Cause Analysis (RCA) published       │
│                                                                       │
│ 2. Planned maintenance 🔧                                         │
│    ├── Upcoming Azure maintenance that may affect you            │
│    ├── Example: "VM host maintenance scheduled next Tuesday"    │
│    ├── Usually zero impact (live migration)                      │
│    └── Get notified so you can plan accordingly                 │
│                                                                       │
│ 3. Health advisories ℹ️                                            │
│    ├── Azure feature changes, retirements, deprecations         │
│    ├── Example: "Classic VMs retiring by Sept 2025"             │
│    ├── Security advisories                                       │
│    └── Action recommended but not urgent                        │
│                                                                       │
│ Service Health vs Status page:                                      │
│ ├── status.azure.com → Public, shows ALL Azure issues          │
│ └── Service Health → Personalized, shows only YOUR stuff       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Configuring Health Alerts (Portal Walkthrough)

```
Service Health → Health alerts → [+ Create service health alert]

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE HEALTH ALERT                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Condition ──                                                     │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│                                                                       │
│ Services:                                                            │
│ ☑ App Service  ☑ Virtual Machines  ☑ SQL Database               │
│ ☑ Azure Kubernetes Service  ☑ Storage                           │
│ ⚡ Select only services you actually use!                        │
│                                                                       │
│ Regions:                                                             │
│ ☑ Central India  ☑ East US                                       │
│                                                                       │
│ Event types:                                                         │
│ ☑ Service issue (outages)                                        │
│ ☑ Planned maintenance                                             │
│ ☑ Health advisory                                                  │
│ ☐ Security advisory                                               │
│                                                                       │
│ ── Action Group ──                                                  │
│ [ag-ops-team]                                                       │
│ Notifications: Email + SMS + Slack webhook                        │
│                                                                       │
│ ── Alert Rule Details ──                                            │
│ Name: [Azure Health - Production Services]                        │
│                                                                       │
│ [Create alert rule]                                                  │
│                                                                       │
│ ⚡ Set this up FIRST when starting with Azure!                   │
│ ⚡ Know immediately when Azure has an issue affecting you.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Azure Advisor

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE ADVISOR                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Advisor                                                  │
│                                                                       │
│ Azure Advisor = Your personal Azure consultant (free!)            │
│ Analyzes your resources and gives recommendations in 5 areas:    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ 💰 Cost         Save money                                   │  │
│ │    ├── Right-size underutilized VMs                        │  │
│ │    ├── Buy reserved instances (save 30-72%)               │  │
│ │    ├── Delete unused public IPs, disks, gateways          │  │
│ │    └── Use Azure Hybrid Benefit (reuse Windows licenses)  │  │
│ │                                                              │  │
│ │ 🔒 Security     Protect your resources                      │  │
│ │    ├── Enable MFA for admin accounts                      │  │
│ │    ├── Fix NSG rules that are too permissive              │  │
│ │    ├── Enable Defender for Cloud                          │  │
│ │    └── Rotate storage account keys                        │  │
│ │                                                              │  │
│ │ ⚡ Reliability   Improve availability                       │  │
│ │    ├── Add redundancy (zone-redundant deployments)        │  │
│ │    ├── Enable Azure Backup                                │  │
│ │    ├── Use availability sets/zones for VMs                │  │
│ │    └── Configure disaster recovery                        │  │
│ │                                                              │  │
│ │ 🚀 Performance  Improve speed                               │  │
│ │    ├── Upgrade to SSD managed disks                       │  │
│ │    ├── Resize to appropriate VM sizes                     │  │
│ │    ├── Enable caching on SQL databases                    │  │
│ │    └── Use CDN for static content                         │  │
│ │                                                              │  │
│ │ 🔧 Operational Excellence  Best practices                   │  │
│ │    ├── Set up health alerts                                │  │
│ │    ├── Enable diagnostic settings                         │  │
│ │    ├── Tag all resources                                   │  │
│ │    └── Use latest API versions                            │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Advisor Score: 0-100% per category                                 │
│ Higher score = Better following best practices                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Advisor Recommendations (Portal Walkthrough)

```
Console → Advisor → (click any category)

┌─────────────────────────────────────────────────────────────────────┐
│           ADVISOR RECOMMENDATIONS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cost Recommendations:                                                │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────┐    │
│ │ Impact: High  💰 Estimated savings: $120/month             │    │
│ │                                                              │    │
│ │ Right-size or shutdown underutilized virtual machines      │    │
│ │                                                              │    │
│ │ VM: myapp-vm (Standard_D4s_v3)                             │    │
│ │ Average CPU over 14 days: 5%                               │    │
│ │ Average memory over 14 days: 12%                           │    │
│ │                                                              │    │
│ │ Recommendation: Resize to Standard_B2s                    │    │
│ │ Savings: $120/month                                         │    │
│ │                                                              │    │
│ │ [Resize VM]  [Postpone]  [Dismiss]                        │    │
│ └────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────┐    │
│ │ Impact: Medium  💰 Estimated savings: $50/month            │    │
│ │                                                              │    │
│ │ Delete unused public IP addresses                          │    │
│ │                                                              │    │
│ │ Public IP: pip-old-lb (not associated with any resource)  │    │
│ │                                                              │    │
│ │ [Delete]  [Postpone]  [Dismiss]                           │    │
│ └────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Actions per recommendation:                                         │
│ ├── Apply (fix the issue directly from Advisor)                 │
│ ├── Postpone (remind me later)                                   │
│ └── Dismiss (I know, it's intentional)                          │
│                                                                       │
│ Advisor Alerts:                                                      │
│ Advisor → Alerts → [+ Create]                                   │
│ Get notified when new recommendations appear!                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Azure Resource Health

```
┌─────────────────────────────────────────────────────────────────────┐
│           RESOURCE HEALTH                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Any resource → Help → Resource health                             │
│ OR: Service Health → Resource health                              │
│                                                                       │
│ Status:                                                              │
│ ├── 🟢 Available: Resource is healthy                            │
│ ├── 🟡 Degraded: Performance issues detected                    │
│ ├── 🔴 Unavailable: Resource is down                             │
│ └── ⚪ Unknown: No health data available                         │
│                                                                       │
│ Shows:                                                               │
│ ├── Current health status                                        │
│ ├── Health history (last 30 days)                                │
│ ├── Root cause (platform vs user-initiated)                     │
│ └── Recommended actions                                          │
│                                                                       │
│ vs Service Health:                                                   │
│ ├── Service Health: Azure platform issues (region-wide)         │
│ └── Resource Health: Individual resource health (your specific VM)│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: az CLI Reference

```bash
# View Advisor recommendations
az advisor recommendation list --output table

# View cost recommendations only
az advisor recommendation list --category Cost --output table

# View security recommendations
az advisor recommendation list --category Security --output table

# Get Advisor score
az advisor recommendation list --category Cost --output json | jq length

# Suppress (dismiss) a recommendation
az advisor recommendation disable \
  --ids /subscriptions/.../recommendations/<recommendation-id>

# View Service Health events
az rest --method get \
  --url "https://management.azure.com/subscriptions/{sub-id}/providers/Microsoft.ResourceHealth/events?api-version=2022-10-01"

# Check resource health
az resource show \
  --ids /subscriptions/.../providers/Microsoft.Compute/virtualMachines/myvm/providers/Microsoft.ResourceHealth/availabilityStatuses/current \
  --api-version 2022-10-01
```

---

## Real-World Patterns

### Pattern 1: Proactive Incident Communication

```
┌─────────────────────────────────────────────────┐
│  Azure Outage → Notification Flow               │
├─────────────────────────────────────────────────┤
│                                                 │
│  Service Health Alert                           │
│  ("App Service issue in East US")               │
│       │                                         │
│       ▼                                         │
│  Action Group                                   │
│  ├── Email: ops-team@company.com               │
│  ├── SMS: On-call engineer                     │
│  ├── Teams webhook: #incidents channel         │
│  └── Logic App: Create PagerDuty incident      │
│                                                 │
│  Advisor Recommendation:                        │
│  "Enable zone redundancy for App Service"       │
│  Impact: Survive single datacenter failure      │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Service Health: Azure outage notifications for YOUR services
├── Service Issues: Current outages affecting you
├── Planned Maintenance: Upcoming maintenance windows
├── Health Advisories: Service changes/retirements
├── Security Advisories: Security-related notifications
└── Health History: Past incidents

Azure Advisor: Personalized recommendations
├── Reliability: HA, backups, zone redundancy
├── Security: Vulnerabilities, misconfigurations
├── Performance: Slow queries, right-sizing
├── Cost: Unused resources, reservations
└── Operational Excellence: Best practices

Set up health alerts → Action Groups → Email/SMS/Teams
Review Advisor weekly → Act on High Impact recommendations
Resource Health: Check individual resource status
```

---

## What's Next?

Next chapter: [Chapter 42: Microsoft Entra ID](42-entra-id.md) — Identity and access management with Azure Active Directory (now Microsoft Entra ID).
│  Service Health Alert                           │
│  ("App Service issue in East US")               │
│       │                                         │
│       ▼                                         │
│  Action Group                                   │
│  ├── Email: ops-team@company.com               │
│  ├── SMS: On-call engineer                     │
│  ├── Teams webhook: #incidents channel         │
│  └── Logic App: Create PagerDuty incident      │
│                                                 │
│  Advisor Recommendation:                        │
│  "Enable zone redundancy for App Service"       │
│  Impact: Survive single datacenter failure      │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure Service Health:
  Service Issues = Azure outages affecting your resources
  Planned Maintenance = Upcoming Azure maintenance
  Health Advisories = Feature retirements, deprecations
  ⚡ Set up health alerts FIRST when starting with Azure!

Azure Advisor (free personal consultant):
  Cost = Save money (right-size VMs, delete unused resources)
  Security = Fix vulnerabilities (MFA, NSG rules)
  Reliability = Improve uptime (zones, backup, DR)
  Performance = Improve speed (SSD, caching)
  Operational Excellence = Best practices (tags, diagnostics)

Resource Health:
  Per-resource health status (Available/Degraded/Unavailable)
  Shows if issue is Azure platform vs your configuration

Advisor Score: 0-100% per category (higher = better)
```

---

## What's Next?

Next chapter: [Chapter 42: Microsoft Entra ID](42-entra-id.md) — Identity and access management with Azure Active Directory (now Microsoft Entra ID).
