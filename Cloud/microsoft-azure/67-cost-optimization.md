# Chapter 67: Cost Optimization Strategies

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Understanding Azure Costs](#part-1-understanding-azure-costs)
- [Part 2: Azure Cost Management (Portal Walkthrough)](#part-2-azure-cost-management-portal-walkthrough)
- [Part 3: Compute Savings](#part-3-compute-savings)
- [Part 4: Storage & Database Savings](#part-4-storage--database-savings)
- [Part 5: Architecture-Level Savings](#part-5-architecture-level-savings)
- [Part 6: Governance & Automation](#part-6-governance--automation)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure cost optimization is about getting the most value from every dollar spent. This chapter covers practical strategies to reduce your Azure bill without sacrificing performance or reliability.

```
What you'll learn:
├── Understanding Azure Costs
├── Azure Cost Management (Portal)
├── Compute Savings (VMs, App Service, AKS)
├── Storage & Database Savings
├── Architecture-Level Savings
├── Governance & Automation
└── Quick reference
```

---

## Part 1: Understanding Azure Costs

```
Where does the money go?
├── Compute (VMs, App Service, AKS): ~60-70% of typical bill
├── Storage (Blob, Disks, Backups): ~10-15%
├── Databases (SQL, Cosmos, Redis): ~10-15%
├── Networking (Bandwidth, VPN, LB): ~5-10%
└── Other (Monitoring, AI, etc.): ~5%

Common cost mistakes:
├── ❌ Over-provisioned VMs (paying for 8 cores, using 2)
├── ❌ Dev/test VMs running 24/7 (nights + weekends wasted)
├── ❌ Orphaned resources (disks, IPs, LBs from deleted VMs)
├── ❌ Premium storage for non-critical workloads
├── ❌ No reservations for predictable workloads
├── ❌ No auto-scaling (paying for peak capacity always)
├── ❌ Not using Azure Hybrid Benefit (existing licenses)
└── ❌ Ignoring Advisor recommendations

Azure Pricing Calculator:
├── https://azure.microsoft.com/pricing/calculator
├── Estimate costs BEFORE deploying
├── Compare different configurations
└── Export and share estimates
```

---

## Part 2: Azure Cost Management (Portal Walkthrough)

```
Console → Cost Management

┌─────────────────────────────────────────────────────────────────────┐
│           COST MANAGEMENT                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cost Analysis:                                                       │
│ ├── View costs by resource, resource group, service, location    │
│ ├── Daily, monthly, yearly trends                                 │
│ ├── Forecast future costs                                         │
│ ├── Filter and drill down                                         │
│ └── Export to CSV                                                  │
│                                                                       │
│ Budgets:                                                             │
│ ├── Set monthly budget (e.g., $500/month)                        │
│ ├── Alert at thresholds (50%, 80%, 100%)                        │
│ ├── Action groups (email, SMS, webhook)                          │
│ ├── Auto-actions: Shutdown VMs when budget exceeded              │
│ └── Create: Cost Management → Budgets → [+ Add]                │
│                                                                       │
│ Advisor Recommendations:                                             │
│ ├── Right-size or shutdown underutilized VMs                     │
│ ├── Purchase reservations (projected savings shown)              │
│ ├── Delete orphaned resources                                     │
│ └── Review weekly!                                                │
│                                                                       │
│ Cost Alerts:                                                         │
│ ├── Budget alerts (approaching/exceeding budget)                 │
│ ├── Credit alerts (EA/MCA approaching credit limit)              │
│ ├── Department spending alerts                                    │
│ └── Anomaly alerts (unusual spending patterns)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Compute Savings

```
1. Reserved Instances (RIs):
   ├── Commit to 1 or 3 years → Save 40-72%
   ├── Apply to: VMs, SQL Database, Cosmos DB, App Service, Redis
   ├── Example: D4s_v5 VM
   │   Pay-as-you-go: ~$280/month
   │   1-year RI: ~$180/month (36% savings)
   │   3-year RI: ~$115/month (59% savings)
   ├── Can exchange or cancel (with fee)
   └── Buy: Console → Reservations → [+ Purchase]

2. Savings Plans:
   ├── Commit to $/hour spend (more flexible than RIs)
   ├── Applies across VM sizes, regions, services
   ├── 1 or 3 year terms
   └── Best for: Varied compute workloads

3. Spot VMs:
   ├── Unused Azure capacity at up to 90% discount!
   ├── Can be evicted with 30-second notice
   ├── Good for: Batch processing, CI/CD, dev/test, ML training
   ├── Not for: Production web servers, databases
   └── Set max price or use -1 for current spot price

4. Azure Hybrid Benefit:
   ├── Bring existing Windows Server / SQL Server licenses
   ├── Save up to 85% on Windows VMs
   ├── Save up to 55% on Azure SQL
   ├── Enable: VM → Configuration → Azure Hybrid Benefit: Yes
   └── Works with: Software Assurance or CSP licenses

5. Auto-shutdown & Auto-scale:
   ├── Dev/test VMs: Auto-shutdown at 7 PM → Save 60%+
   │   VM → Auto-shutdown → Enable → 19:00 → Save
   ├── App Service: Scale down at night (2 instances → 1)
   ├── AKS: Cluster auto-scaler + node pool scale to 0
   └── VMSS: Schedule-based scaling profiles

6. Right-sizing:
   ├── Azure Advisor → "Right-size or shutdown underutilized VMs"
   ├── Check: VM → Metrics → CPU average < 5%? Downsize!
   ├── Example: D4s_v5 (4 vCPU) → D2s_v5 (2 vCPU) = 50% savings
   └── Review monthly
```

---

## Part 4: Storage & Database Savings

```
Storage:
├── Access tiers: Move cold data to Cool (-50%) or Archive (-90%)
│   Blob → Change tier → Cool / Archive
├── Lifecycle Management: Auto-move blobs between tiers
│   "After 30 days → Cool, after 90 days → Archive"
├── Delete old snapshots and versions
├── Use Standard HDD for non-critical disks (dev/test)
├── Resize over-provisioned disks
└── Enable soft delete but set reasonable retention (7 days, not 365)

Database:
├── Azure SQL:
│   ├── Serverless: Auto-pause after idle (pay $0 when paused)
│   ├── Elastic Pools: Share resources across databases
│   ├── Reserved capacity: 1-3 year commitment
│   └── DTU vs vCore: Calculate which is cheaper
│
├── Cosmos DB:
│   ├── Autoscale: Scale down during low usage
│   ├── Serverless: Great for dev/test (pay per request)
│   ├── Reserved capacity: 1-3 year for provisioned RUs
│   └── TTL: Auto-delete old documents
│
├── Redis:
│   ├── Use Basic/Standard for dev/test (not Premium)
│   └── Right-size: Monitor memory usage
│
└── General:
    ├── Read replicas only if needed
    ├── Geo-replication only if needed (DR)
    └── Review backup retention (default is often generous)
```

---

## Part 5: Architecture-Level Savings

```
1. Serverless first:
   ├── Azure Functions (Consumption): Pay per execution
   ├── Container Apps: Scale to zero
   ├── Logic Apps (Consumption): Pay per action
   └── $0 when not running!

2. PaaS over IaaS:
   ├── App Service instead of VM + IIS: Less management, often cheaper
   ├── Azure SQL instead of SQL Server on VM: No OS patching
   └── Managed services reduce hidden operations cost

3. Caching:
   ├── Redis cache reduces expensive DB queries
   ├── CDN caches static content (reduces bandwidth costs)
   └── API caching in APIM (reduce backend calls)

4. Async processing:
   ├── Queue expensive operations (don't use premium compute for waiting)
   ├── Process in batch (cheaper than real-time)
   └── Use Spot VMs for batch workers

5. Multi-tenancy:
   ├── Share infrastructure across customers/teams
   ├── AKS: Multiple apps per cluster
   ├── App Service Plan: Multiple apps per plan
   └── SQL Elastic Pool: Multiple DBs sharing resources

6. Dev/Test subscriptions:
   ├── Azure Dev/Test pricing: Discounted VM rates
   ├── No Windows license charges on dev/test
   ├── Free Microsoft software for development
   └── Create separate subscription for dev/test
```

---

## Part 6: Governance & Automation

```
Tagging strategy:
├── Tag ALL resources:
│   Environment: dev/staging/prod
│   CostCenter: engineering/marketing/ops
│   Owner: team-name or email
│   Project: project-name
├── Azure Policy: "Require CostCenter tag on all resources"
├── Cost analysis: Filter by tag → See cost per team/project
└── Chargeback: Bill teams for their actual usage

Automation:
├── Azure Automation runbooks:
│   "Every Friday at 8 PM: Stop all dev VMs"
│   "Every Monday at 8 AM: Start all dev VMs"
├── Azure Functions: Custom cost optimization logic
├── Azure Policy: Deny expensive VM sizes in dev subscription
└── Budget alerts + action groups: Auto-respond to overspend

Regular reviews:
├── Weekly: Check Azure Advisor recommendations
├── Monthly: Review Cost Analysis trends
├── Quarterly: Review reservations and right-sizing
├── Annually: Renew/adjust reservations
└── Always: Tag new resources, delete unused ones

Cost optimization checklist:
☐ Set budgets and alerts for all subscriptions
☐ Review Advisor recommendations weekly
☐ Right-size underutilized VMs
☐ Auto-shutdown dev/test VMs
☐ Purchase reservations for stable workloads
☐ Use Spot VMs for fault-tolerant batch jobs
☐ Enable Azure Hybrid Benefit
☐ Lifecycle management for blob storage
☐ Tag all resources for cost attribution
☐ Delete orphaned resources (disks, IPs, snapshots)
```

---

## Part 7: az CLI Reference

```bash
# --- Cost Management ---

# View current costs (this month)
az costmanagement query --type ActualCost \
  --scope "subscriptions/<sub-id>" \
  --timeframe MonthToDate \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  -o table

# View costs grouped by resource group
az costmanagement query --type ActualCost \
  --scope "subscriptions/<sub-id>" \
  --timeframe MonthToDate \
  --dataset-grouping name=ResourceGroup type=Dimension

# Create a budget
az consumption budget create \
  --budget-name monthly-budget \
  --amount 500 \
  --time-grain Monthly \
  --start-date 2024-01-01 \
  --end-date 2025-12-31 \
  --resource-group rg-prod

# List budgets
az consumption budget list -o table

# --- Reservations ---

# List available reservations
az reservations catalog show \
  --subscription-id <sub-id> \
  --reserved-resource-type VirtualMachines \
  --location centralindia

# List purchased reservations
az reservations reservation-order list -o table

# --- Advisor Recommendations ---

# List cost recommendations
az advisor recommendation list --category Cost -o table

# --- Resource Cleanup ---

# Find unattached managed disks (orphaned)
az disk list --query "[?managedBy==null].{Name:name, RG:resourceGroup, Size:diskSizeGb}" -o table

# Find unattached public IPs
az network public-ip list --query "[?ipConfiguration==null].{Name:name, RG:resourceGroup}" -o table

# Find stopped VMs still incurring disk costs
az vm list -d --query "[?powerState=='VM deallocated'].{Name:name, RG:resourceGroup}" -o table

# Auto-shutdown a VM (set for 7 PM daily)
az vm auto-shutdown \
  --name vm-dev-01 \
  --resource-group rg-dev \
  --time 1900

# Resize a VM (right-size)
az vm resize \
  --name vm-overprovisioned \
  --resource-group rg-prod \
  --size Standard_D2s_v5

# Enable Azure Hybrid Benefit on a VM
az vm update \
  --name vm-windows \
  --resource-group rg-prod \
  --set licenseType=Windows_Server

# Tag resources for cost attribution
az resource tag \
  --tags CostCenter=Engineering Environment=Production \
  --ids /subscriptions/<sub-id>/resourceGroups/rg-prod

# Export cost data
az costmanagement export create \
  --name daily-export \
  --scope "subscriptions/<sub-id>" \
  --type ActualCost \
  --timeframe MonthToDate \
  --storage-account-id /subscriptions/<sub-id>/resourceGroups/rg-billing/providers/Microsoft.Storage/storageAccounts/stexports \
  --storage-container exports \
  --schedule-recurrence Daily
```

---

## Quick Reference

```
Top Cost Savings Strategies:

1. Reserved Instances: Save 40-72% (1-3 year commit)
2. Spot VMs: Save up to 90% (evictable, batch jobs)
3. Azure Hybrid Benefit: Save 55-85% (bring your licenses)
4. Auto-shutdown: Save 60%+ (stop dev/test at night)
5. Right-sizing: Save 30-50% (downsize underutilized VMs)
6. Serverless: $0 when idle (Functions, Container Apps)
7. Storage tiers: Save 50-90% (Cool/Archive for cold data)
8. SQL Serverless: Auto-pause when idle

Tools:
├── Cost Management: Analysis, budgets, alerts
├── Azure Advisor: Free recommendations
├── Pricing Calculator: Estimate before deploying
├── Reservations: Purchase commitments
└── Tags: Cost attribution per team/project

Review cadence:
├── Weekly: Advisor recommendations
├── Monthly: Cost trends, orphaned resources
├── Quarterly: Right-sizing, reservations
└── Tag everything, delete unused resources!
```

---

## What's Next?

Next chapter: [Chapter 68: Disaster Recovery & BCDR](68-disaster-recovery.md) — Protect your business with backup, recovery, and business continuity planning.
