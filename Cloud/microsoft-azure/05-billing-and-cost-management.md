# Chapter 5: Billing & Cost Management (Azure)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Pricing Fundamentals](#part-1-azure-pricing-fundamentals)
- [Part 2: Pricing Models](#part-2-pricing-models)
- [Part 3: Free Tier & Free Services](#part-3-free-tier--free-services)
- [Part 4: Billing Hierarchy](#part-4-billing-hierarchy)
- [Part 5: Cost Management + Billing Portal](#part-5-cost-management--billing-portal)
- [Part 6: Cost Analysis (Azure's Cost Explorer)](#part-6-cost-analysis-azures-cost-explorer)
- [Part 7: Budgets & Alerts](#part-7-budgets--alerts)
- [Part 8: Azure Advisor (Cost Recommendations)](#part-8-azure-advisor-cost-recommendations)
- [Part 9: Purchasing Reservations](#part-9-purchasing-reservations)
- [Part 10: Azure Pricing Calculator & TCO Calculator](#part-10-azure-pricing-calculator--tco-calculator)
- [Part 11: Cost Optimization Strategies](#part-11-cost-optimization-strategies)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference: Console Navigation](#quick-reference-console-navigation)
- [What's Next?](#whats-next)

---

## Overview

Azure cost management involves understanding pricing models, using Cost Management + Billing tools, setting budgets, and optimizing spend with reservations and hybrid benefits. Azure's billing is tied to subscriptions and Enterprise Agreements.

```
What you'll learn:
├── Azure pricing fundamentals
├── Pricing models (Pay-As-You-Go, Reserved, Spot, Savings Plans)
├── Azure Hybrid Benefit (unique to Azure!)
├── Free Tier & Free Services
├── Billing hierarchy (EA, MCA, CSP, PAYG)
├── Cost Management + Billing portal walkthrough
├── Cost Analysis (the cost explorer equivalent)
├── Budgets & Alerts
├── Cost allocation & tags
├── Reserved Instances & Azure Savings Plans
├── Spot VMs
├── Azure Advisor cost recommendations
├── Azure Pricing Calculator & TCO Calculator
├── Cost optimization strategies
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: Azure Pricing Fundamentals

### How Azure Charges You

```
┌─────────────────────────────────────────────────────────────────────┐
│                   AZURE PRICING PRINCIPLES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. PAY-AS-YOU-GO (consumption-based)                                │
│     └── VM per minute (rounded), storage per GB-month               │
│     └── Most services bill per second or per minute                 │
│                                                                       │
│  2. RESERVED INSTANCES / SAVINGS PLANS (commitment discounts)        │
│     └── 1 or 3 year commitment → up to 72% discount                │
│                                                                       │
│  3. AZURE HYBRID BENEFIT (unique advantage!)                         │
│     └── Use existing Windows Server or SQL Server licenses on Azure │
│     └── Save up to 85% on Windows VMs (combined with RI)           │
│     └── ⚡ Major differentiator vs AWS/GCP                           │
│                                                                       │
│  4. TIERED PRICING (volume discounts)                                │
│     └── Blob Storage: Hot first 50 TB cheaper than next 450 TB     │
│     └── Bandwidth: first 5 GB free, then tiered                    │
│                                                                       │
│  5. DATA TRANSFER                                                    │
│     └── Inbound: FREE (almost always)                               │
│     └── Outbound to internet:                                       │
│         ├── First 100 GB/month: FREE                               │
│         ├── 100 GB - 10 TB: ~$0.087/GB                             │
│         └── 10 TB+: tiered pricing (decreasing)                     │
│     └── Between regions: $0.02-0.08/GB (varies by pair)            │
│     └── Within same region/AZ: FREE (via private IP)               │
│                                                                       │
│  Billing cycle: Monthly (calendar month)                             │
│  Payment: Credit card, invoice (EA/MCA), bank transfer              │
│  Currency: Multiple currencies supported                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### What Costs Money

```
┌──────────────────────────────────────────────────────────────────────┐
│ Service           │ What You Pay For                                  │
├──────────────────────────────────────────────────────────────────────┤
│ Virtual Machines  │ Per minute, based on VM size + OS. Stopped        │
│                   │ (deallocated) = no compute, still pay for disks  │
│                   │ ⚠️ "Stopped" in portal ≠ deallocated! Must       │
│                   │ "Stop (deallocate)" to stop billing              │
│                   │                                                    │
│ Managed Disks     │ Per GB-month provisioned (not used!)             │
│                   │ Premium SSD: ~$0.132/GB, Standard SSD: ~$0.048  │
│                   │ Standard HDD: ~$0.04                              │
│                   │                                                    │
│ Blob Storage      │ Per GB-month stored + operations + egress        │
│                   │ Hot: $0.018/GB, Cool: $0.010/GB                  │
│                   │ Cold: $0.0036/GB, Archive: $0.00099/GB           │
│                   │                                                    │
│ Azure SQL         │ DTU model: Per DTU level/hour                    │
│                   │ vCore model: Per vCore/hour + storage            │
│                   │ Serverless: Auto-pause (pay when active)         │
│                   │                                                    │
│ App Service       │ Per App Service Plan tier, not per app           │
│                   │ Free/Shared: Free, Basic: ~$13/month             │
│                   │ Standard: ~$70/month, Premium: ~$116/month       │
│                   │ 💡 Multiple apps on one plan = same price         │
│                   │                                                    │
│ Azure Functions   │ Consumption: Per execution + GB-seconds          │
│                   │ First 1M executions/month FREE                   │
│                   │ Premium: Per vCPU-second (always-warm instances) │
│                   │                                                    │
│ AKS               │ Control plane: FREE! (unlike AWS/GCP!)          │
│                   │ Pay only for nodes (VMs) + disks + networking    │
│                   │ 💡 AKS is one of the most cost-effective K8s     │
│                   │                                                    │
│ Azure SQL         │ Serverless: Auto-pause after idle period         │
│ (Serverless)      │ Only pay when queries run (great for dev!)       │
│                   │                                                    │
│ Load Balancer     │ Basic: FREE! Standard: per rule + per GB        │
│                   │                                                    │
│ NAT Gateway       │ Per hour (~$0.045) + per GB (~$0.045)           │
│                   │                                                    │
│ Azure Front Door  │ Per routing rule/day + per GB + per request     │
│                   │                                                    │
│ Key Vault         │ $0.03/10K operations (secrets/keys)              │
│                   │ HSM keys: $1/key/month                           │
│                   │                                                    │
│ Azure Monitor     │ Logs: $2.76/GB ingested (first 5 GB free)       │
│                   │ ⚠️ Can get very expensive!                        │
│                   │ Metrics: First 150 metrics free                  │
│                   │                                                    │
│ Cosmos DB         │ Per RU/second provisioned or serverless (per RU) │
│                   │ Serverless: $0.25/million RUs (great for low use)│
│                   │                                                    │
│ Public IP         │ Basic: FREE, Standard: ~$0.004/hour             │
│                   │ ⚠️ Static Standard IP = ~$3.65/month even unused │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Pricing Models

### All Azure Pricing Options

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PRICING MODELS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. PAY-AS-YOU-GO (PAYG — full price, no commitment)                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Pay per minute/second based on VM size                   │   │
│  │ ├── No commitment, cancel anytime                            │   │
│  │ ├── Most expensive per unit                                  │   │
│  │ ├── Best for: Testing, short-term, unpredictable workloads   │   │
│  │ │                                                             │   │
│  │ │ Example: D2s v5 in Central India                           │   │
│  │ │ Linux: ~$0.096/hour ≈ $70/month                            │   │
│  │ │ Windows: ~$0.192/hour ≈ $140/month (2x for Windows!)      │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  2. RESERVED INSTANCES (1 or 3 year commitment)                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Commit to specific VM size + region                      │   │
│  │ ├── 1 year: Up to 40% discount                               │   │
│  │ ├── 3 years: Up to 65% discount                              │   │
│  │ ├── Payment: All Upfront or Monthly                          │   │
│  │ ├── Can exchange (change VM size within same family)         │   │
│  │ │   → More flexible than AWS Standard RIs!                   │   │
│  │ ├── Can cancel (with early termination fee of 12%)           │   │
│  │ │   → ⚡ More flexible than AWS (cannot cancel)              │   │
│  │ ├── Instance size flexibility: Within same series/region     │   │
│  │ │   D2s v5 RI can cover 2x D1s v5 or 0.5x D4s v5           │   │
│  │ │                                                             │   │
│  │ │ Example: D2s v5, 1yr RI, Central India                    │   │
│  │ │ Price: ~$0.058/hr ≈ $42/month (40% savings)               │   │
│  │ │                                                             │   │
│  │ │ Example: D2s v5, 3yr RI, Central India                    │   │
│  │ │ Price: ~$0.038/hr ≈ $28/month (60% savings)               │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  3. AZURE SAVINGS PLANS (flexible commitment — NEWER)                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Commit to $/hour spend (not instance type)               │   │
│  │ ├── 1 year or 3 years                                        │   │
│  │ ├── Up to 65% discount                                       │   │
│  │ ├── Two types:                                               │   │
│  │ │   ├── Compute: Any VM, any region, any OS                 │   │
│  │ │   │   (Also covers: App Service, Functions Premium,       │   │
│  │ │   │    Container Instances)                                │   │
│  │ │   └── Machine type: Specific VM family + region           │   │
│  │ │       (Slightly higher discount, less flexible)           │   │
│  │ ├── ⚡ Similar to AWS Savings Plans                           │   │
│  │ ├── Can be combined with Azure Hybrid Benefit!              │   │
│  │ │                                                             │   │
│  │ │ Example: Commit $5/hour, 1 year                           │   │
│  │ │ → All compute up to $5/hr gets discounted rate            │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  4. SPOT VMs (up to 90% discount, can be evicted)                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Up to 90% discount vs PAYG                               │   │
│  │ ├── ⚠️ Can be evicted with 30-second warning                 │   │
│  │ ├── You set max price (or use -1 for market price)           │   │
│  │ ├── Eviction policy: Stop/Deallocate or Delete               │   │
│  │ ├── Best for: Batch, CI/CD, stateless, interruptible         │   │
│  │ ├── NOT for: Databases, production single-instance           │   │
│  │ │                                                             │   │
│  │ │ Example: D2s v5 Spot in Central India                      │   │
│  │ │ Price: ~$0.019/hr ≈ $14/month (80% savings)               │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  5. AZURE HYBRID BENEFIT (bring existing licenses!)                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Use existing Windows Server licenses on Azure VMs       │   │
│  │ │   → Save up to 40% on Windows VMs                          │   │
│  │ │   → Combined with RI: Save up to 80%!                     │   │
│  │ │                                                             │   │
│  │ ├── Use existing SQL Server licenses on Azure SQL           │   │
│  │ │   → Save up to 55% on Azure SQL                            │   │
│  │ │   → Combined with RI: Save up to 85%!                     │   │
│  │ │                                                             │   │
│  │ ├── Also available for:                                      │   │
│  │ │   ├── Red Hat Linux subscriptions                         │   │
│  │ │   └── SUSE Linux subscriptions                            │   │
│  │ │                                                             │   │
│  │ ├── Eligible: Software Assurance or equivalent licenses     │   │
│  │ │                                                             │   │
│  │ │ Example: D2s v5 Windows, 3yr RI + Hybrid Benefit          │   │
│  │ │ PAYG:                     ~$140/month                      │   │
│  │ │ With 3yr RI:              ~$56/month (60% off)             │   │
│  │ │ With RI + Hybrid Benefit: ~$28/month (80% off!)            │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  COMPARISON:                                                         │
│  ┌────────────────┬─────────┬──────────┬──────────┬──────────────┐ │
│  │ Model          │ Discount│ Commit   │ Flexible │ Interruptible│ │
│  ├────────────────┼─────────┼──────────┼──────────┼──────────────┤ │
│  │ PAYG           │ 0%      │ None     │ Full     │ No           │ │
│  │ Reserved (1yr) │ 30-40%  │ 1 year   │ Medium   │ No           │ │
│  │ Reserved (3yr) │ 50-65%  │ 3 years  │ Medium   │ No           │ │
│  │ Savings Plan   │ 30-65%  │ 1-3 yr   │ High     │ No           │ │
│  │ Spot           │ 60-90%  │ None     │ Full     │ YES          │ │
│  │ Hybrid Benefit │ 40-55%  │ License  │ Medium   │ No           │ │
│  │ RI + Hybrid    │ 80-85%  │ 1-3yr+lic│ Medium   │ No           │ │
│  └────────────────┴─────────┴──────────┴──────────┴──────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Free Tier & Free Services

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AZURE FREE TIER                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. FREE ACCOUNT ($200 credit for 30 days)                           │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── $200 credit for first 30 days                            │    │
│ │ ├── 12 months of popular free services                       │    │
│ │ ├── 65+ always-free services                                 │    │
│ │ └── Upgrade to PAYG to continue after 30 days               │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 2. 12-MONTH FREE (after account creation)                           │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Virtual Machines:  750 hrs/month B1S (Linux + Windows)       │    │
│ │ Managed Disks:     2x 64 GB P6 SSD                         │    │
│ │ Blob Storage:      5 GB LRS Hot, 20K read, 10K write ops   │    │
│ │ Azure SQL:         250 GB S0 database                       │    │
│ │ Cosmos DB:         1,000 RU/s + 25 GB                      │    │
│ │ Azure Files:       5 GB LRS                                 │    │
│ │ Bandwidth:         15 GB outbound                           │    │
│ │ App Service:       10 web/mobile/API apps (F1 tier)         │    │
│ │ Azure Functions:   1M requests/month                        │    │
│ │ Container Registry:100 GB (Basic)                           │    │
│ │ AI services:       Various free amounts                     │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 3. ALWAYS FREE (never expires)                                       │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ AKS:               Control plane FREE (pay only for nodes)  │    │
│ │ Azure DevOps:      5 users + 1 free parallel job            │    │
│ │ Azure Functions:   1M requests/month (Consumption plan)     │    │
│ │ Event Grid:        100K operations/month                    │    │
│ │ Advisor:           FREE (cost optimization recommendations) │    │
│ │ Microsoft Defender: Free tier (basic recommendations)       │    │
│ │ Security Center:   Free tier (basic CSPM)                   │    │
│ │ Azure Policy:      FREE (governance and compliance)         │    │
│ │ Cost Management:   FREE (all cost analysis tools)           │    │
│ │ Azure Monitor:     First 5 GB log data/month                │    │
│ │ Load Balancer:     Basic SKU: FREE                          │    │
│ │ Service Bus:       Free messaging (Basic tier)              │    │
│ │ Notification Hubs: 1M push notifications                    │    │
│ │ Batch:             FREE (pay only for compute resources)    │    │
│ │ Azure Active Dir:  Free tier (50K objects)                  │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ ⚠️ COMMON TRAPS:                                                     │
│ ├── VM "Stop" vs "Stop (deallocate)" — only deallocate stops billing│
│ ├── Managed Disks charge even when VM is deallocated               │
│ ├── Public IPs charge when not attached                            │
│ ├── Free B1S is only 750 hrs/month (1 VM continuously)             │
│ ├── Azure Monitor Logs can spike fast ($2.76/GB!)                  │
│ └── Storage transactions add up with frequent access               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Billing Hierarchy

### Azure Agreement Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                  BILLING ACCOUNT TYPES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. PAY-AS-YOU-GO (PAYG) — Individuals & Small Companies             │
│    ├── Sign up with credit card                                     │
│    ├── No minimum commitment                                        │
│    ├── One subscription per billing account (can create more)       │
│    └── Best for: Personal, startups, small workloads                │
│                                                                       │
│ 2. MICROSOFT CUSTOMER AGREEMENT (MCA) — Mid-Size Companies          │
│    ├── Digital agreement (replaces Web Direct)                      │
│    ├── Multiple billing profiles (e.g., per department)             │
│    ├── Invoice sections for cost organization                       │
│    ├── Multiple payment methods per profile                         │
│    └── Best for: Growing companies, some enterprise features        │
│                                                                       │
│ 3. ENTERPRISE AGREEMENT (EA) — Large Enterprises                     │
│    ├── 3-year commitment with Microsoft                             │
│    ├── Volume pricing discounts                                     │
│    ├── Azure Prepayment (monetary commitment → discount)            │
│    ├── Departments → Accounts → Subscriptions hierarchy             │
│    ├── Access to EA Portal (ea.azure.com)                           │
│    └── Best for: 500+ users, $100K+/year Azure spend                │
│                                                                       │
│ 4. CSP (Cloud Solution Provider) — Through a Partner                 │
│    ├── Buy Azure through a Microsoft partner                        │
│    ├── Partner manages billing and support                          │
│    ├── May offer bundled services                                   │
│    └── Best for: Companies wanting managed cloud services           │
│                                                                       │
│ EA Hierarchy:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ EA Enrollment                                                 │    │
│ │ ├── Department: Engineering                                  │    │
│ │ │   ├── Account: Prod Workloads                             │    │
│ │ │   │   ├── Subscription: Prod-WebApp                       │    │
│ │ │   │   └── Subscription: Prod-Database                     │    │
│ │ │   └── Account: Dev Workloads                              │    │
│ │ │       └── Subscription: Dev-All                            │    │
│ │ ├── Department: Data Science                                 │    │
│ │ │   └── Account: ML Workloads                                │    │
│ │ │       └── Subscription: DS-Production                     │    │
│ │ └── Department: IT                                           │    │
│ │     └── Account: Shared Services                             │    │
│ │         └── Subscription: Shared-Networking                  │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Cost Management + Billing Portal

### Portal Walkthrough

```
Portal → Cost Management + Billing (search or left menu)

┌─────────────────────────────────────────────────────────────────┐
│             COST MANAGEMENT + BILLING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Left navigation:                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Overview              ← Dashboard summary                  │   │
│ │                                                            │   │
│ │ Cost Management:                                           │   │
│ │   Cost analysis       ← Analyze costs (graphs + tables)   │   │
│ │   Cost alerts         ← View triggered alerts              │   │
│ │   Budgets             ← Create spending limits             │   │
│ │   Advisor recs        ← Cost optimization suggestions      │   │
│ │                                                            │   │
│ │ Billing:                                                   │   │
│ │   Invoices            ← Download monthly invoices          │   │
│ │   Payment methods     ← Credit card, bank account          │   │
│ │   Billing profiles    ← Organization/department billing    │   │
│ │   Products            ← Subscriptions, reservations        │   │
│ │   Azure reservations  ← Buy/manage Reserved Instances      │   │
│ │   Azure savings plans ← Buy/manage Savings Plans          │   │
│ │                                                            │   │
│ │ Settings:                                                  │   │
│ │   Access control      ← Who can see billing data          │   │
│ │   Billing account     ← Account details                    │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ ⚠️ Cost Management is FREE for all Azure customers               │
│    (including Cost analysis, budgets, alerts, export)            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Cost Analysis (Azure's Cost Explorer)

```
Cost Management → Cost analysis

┌─────────────────────────────────────────────────────────────────┐
│                    COST ANALYSIS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Scope: [Subscription: Prod-WebApp ▼]                            │
│ (Can scope to: Management Group, Subscription, Resource Group)  │
│                                                                   │
│ View: [Accumulated costs ▼]                                     │
│ (Options: Accumulated, Daily costs, Cost by service,            │
│  Cost by resource, Invoice details)                              │
│                                                                   │
│ Date range: [This month ▼] (or custom range)                    │
│ Granularity: [Daily ▼] (Daily/Monthly)                          │
│                                                                   │
│ Group by: [Service name ▼]                                      │
│ (Options: Service, Resource, Resource group, Meter category,    │
│  Tag, Location, Subscription)                                    │
│                                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │  $4000 ┤                          ┌────────── Forecast   │   │
│ │        │                     ╱╱╱╱╱│                      │   │
│ │  $3000 ┤                ╱╱╱╱╱     │                      │   │
│ │        │           ╱╱╱╱╱          │                      │   │
│ │  $2000 ┤      ╱╱╱╱╱               │                      │   │
│ │        │  ╱╱╱╱╱                   │                      │   │
│ │  $1000 ┤╱╱                        │                      │   │
│ │        │                          │                      │   │
│ │     $0 └──┬──┬──┬──┬──┬──┬──┬──┬──                      │   │
│ │          1  4  7  10 13 16 19 22 25                      │   │
│ │          May 2026                                        │   │
│ │                                                            │   │
│ │  Budget: $5,000 ─────────────────────── (dashed line)    │   │
│ │  Forecast: $4,150 (end of month)                         │   │
│ │  Actual so far: $2,847                                   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Filters:                                                         │
│ ├── Service name: [All / Virtual Machines / Storage / etc.]     │
│ ├── Resource group: [All / rg-prod-compute / etc.]              │
│ ├── Resource: [All / specific resource name]                    │
│ ├── Location: [All / Central India / etc.]                      │
│ ├── Tag: [Environment: Production]                              │
│ ├── Meter: [Specific meter/SKU]                                 │
│ └── Pricing model: [PAYG / Reservation / Spot / All]            │
│                                                                   │
│ Smart views (pre-built):                                         │
│ ├── Cost by service                                             │
│ ├── Cost by resource                                            │
│ ├── Cost by resource group                                      │
│ ├── DailyCosts (daily trend)                                    │
│ └── Save custom views for reuse                                 │
│                                                                   │
│ [Download] [Share] [Pin to dashboard] [Save as]                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Budgets & Alerts

### Creating a Budget

```
Cost Management → Budgets → Add

┌─────────────────────────────────────────────────────────────────┐
│                    CREATE BUDGET                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Scope: [Subscription: Prod-WebApp ▼]                            │
│                                                                   │
│ Budget details:                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Name:            [prod-monthly-budget]                     │   │
│ │ Reset period:    [Monthly ▼] (Monthly/Quarterly/Annual)   │   │
│ │ Creation date:   [05/2026]                                │   │
│ │ Expiration date: [05/2027]                                │   │
│ │ Amount:          [$5,000]                                  │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Filters (optional — scope the budget):                           │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Resource groups:  [All / Specific RGs]                     │   │
│ │ Resources:        [All / Specific]                         │   │
│ │ Meters:           [All / Specific]                         │   │
│ │ Tags:             [Environment: production]                │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Alert conditions:                                                │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Condition 1:                                               │   │
│ │   Type:       [Actual ▼] (Actual / Forecasted)            │   │
│ │   % of budget: [80]                                       │   │
│ │                                                            │   │
│ │ Condition 2:                                               │   │
│ │   Type:       [Actual]                                    │   │
│ │   % of budget: [100]                                      │   │
│ │                                                            │   │
│ │ Condition 3:                                               │   │
│ │   Type:       [Forecasted]                                │   │
│ │   % of budget: [100]                                      │   │
│ │                                                            │   │
│ │ Alert recipients (email):                                  │   │
│ │ [devops@techcorp.com; finance@techcorp.com]               │   │
│ │                                                            │   │
│ │ Action group (optional — for automation):                  │   │
│ │ [ag-budget-alert ▼]                                       │   │
│ │ → Can trigger: Azure Function, Logic App, webhook,        │   │
│ │   runbook, etc.                                            │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  az consumption budget create \
    --budget-name "prod-monthly-budget" \
    --amount 5000 \
    --category Cost \
    --time-grain Monthly \
    --start-date 2026-05-01 \
    --end-date 2027-05-01 \
    --resource-group rg-prod-compute

  # Note: Alert configuration via CLI is limited; 
  # use Portal or ARM template for full setup
```

---

## Part 8: Azure Advisor (Cost Recommendations)

```
Portal → Advisor → Cost

┌─────────────────────────────────────────────────────────────────┐
│                   AZURE ADVISOR - COST                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Score: 78/100 (cost optimization score)                         │
│                                                                   │
│ Recommendations:                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ 🔵 HIGH IMPACT                                             │   │
│ │                                                            │   │
│ │ 1. Right-size underutilized virtual machines               │   │
│ │    VM: prod-worker-03 (D4s v5 → D2s v5)                  │   │
│ │    CPU avg: 12%, Memory avg: 25%                          │   │
│ │    Potential savings: $52/month                            │   │
│ │    [View resource] [Resize]                                │   │
│ │                                                            │   │
│ │ 2. Buy Reserved Instances for steady-state VMs             │   │
│ │    6 VMs running 24/7 for 90+ days                        │   │
│ │    Recommended: 1-year RI for D2s v5 (6 instances)        │   │
│ │    Potential savings: $1,200/year                          │   │
│ │    [Buy reservations]                                      │   │
│ │                                                            │   │
│ │ 3. Delete unused public IP addresses                       │   │
│ │    3 Standard Public IPs not attached                     │   │
│ │    Potential savings: $11/month                            │   │
│ │    [Delete]                                                │   │
│ │                                                            │   │
│ │ 🟡 MEDIUM IMPACT                                           │   │
│ │                                                            │   │
│ │ 4. Consider Azure Hybrid Benefit                           │   │
│ │    4 Windows VMs eligible for Hybrid Benefit              │   │
│ │    Potential savings: $200/month                           │   │
│ │    [Enable AHB]                                            │   │
│ │                                                            │   │
│ │ 5. Shut down unused virtual machines                       │   │
│ │    VM: staging-test-01 (idle for 14 days)                 │   │
│ │    Potential savings: $70/month                            │   │
│ │    [Stop VM] [Configure auto-shutdown]                     │   │
│ │                                                            │   │
│ │ 6. Review unattached managed disks                        │   │
│ │    2 disks not attached to any VM                         │   │
│ │    Potential savings: $15/month                            │   │
│ │    [Delete disks]                                          │   │
│ │                                                            │   │
│ │ Total potential savings: $1,548/year ($129/month)          │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ ✅ Check Advisor weekly! It's FREE and always has good advice    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Purchasing Reservations

```
Portal → Reservations → Purchase

┌─────────────────────────────────────────────────────────────────┐
│               PURCHASE RESERVATION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Product: [Virtual Machines ▼]                                    │
│ (Also available: SQL Database, Cosmos DB, App Service,          │
│  Azure Synapse, Storage, Redis, Databricks, etc.)               │
│                                                                   │
│ Scope:                                                           │
│ ○ Shared (applies across all subscriptions in billing scope)    │
│ ○ Single subscription: [Prod-WebApp ▼]                          │
│ ○ Single resource group: [rg-prod-compute ▼]                   │
│ ○ Management group                                              │
│                                                                   │
│ Region:         [Central India ▼]                                │
│ VM size:        [D2s_v5 ▼]                                      │
│ Term:           ○ 1 year  ○ 3 years                             │
│ Billing freq:   ○ All Upfront  ○ Monthly                        │
│ Quantity:       [4]                                               │
│                                                                   │
│ Pricing:                                                         │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Pay-As-You-Go:   $0.096/hr × 4 = $0.384/hr              │   │
│ │ 1-year RI:       $0.058/hr × 4 = $0.232/hr (40% off)    │   │
│ │ 3-year RI:       $0.038/hr × 4 = $0.152/hr (60% off)    │   │
│ │                                                            │   │
│ │ Monthly savings (1yr): $111/month                         │   │
│ │ Monthly savings (3yr): $169/month                         │   │
│ │                                                            │   │
│ │ 💡 With Azure Hybrid Benefit (Windows):                    │   │
│ │    Additional 40% off the Windows license cost!            │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Instance size flexibility:                                       │
│ ☑ Enabled (D2s v5 RI can cover 2× D1s or 0.5× D4s in same    │
│   family)                                                        │
│                                                                   │
│ [Review + buy]                                                   │
│                                                                   │
│ ⚡ Unlike AWS: Can EXCHANGE (change VM size/region) and          │
│   CANCEL with early termination fee (12% of remaining)          │
│   → Much more flexible!                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  az reservations reservation-order purchase \
    --reservation-order-id GUID \
    --sku Standard_D2s_v5 \
    --location centralindia \
    --quantity 4 \
    --term P1Y \
    --billing-plan Monthly \
    --applied-scope-type Shared \
    --display-name "prod-d2s-reservation"
```

---

## Part 10: Azure Pricing Calculator & TCO Calculator

### Pricing Calculator

```
URL: azure.microsoft.com/pricing/calculator/

┌─────────────────────────────────────────────────────────────────┐
│               AZURE PRICING CALCULATOR                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Example estimate:                                                │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Service          │ Config                  │ Monthly Cost  │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ VMs (2x)         │ D2s v5, PAYG, Linux     │ $140.00      │   │
│ │ Managed Disks    │ 2× 128 GB P10 SSD       │ $38.00       │   │
│ │ Azure SQL        │ S2 (50 DTU), 250 GB     │ $75.00       │   │
│ │ Blob Storage     │ 100 GB Hot, LRS         │ $1.80        │   │
│ │ App Service      │ S1 Standard             │ $70.00       │   │
│ │ App Gateway      │ Standard v2 + WAF       │ $100.00      │   │
│ │ Azure DNS        │ 1 zone + 1M queries     │ $1.00        │   │
│ │ NAT Gateway      │ 1 gateway + 50 GB       │ $34.50       │   │
│ │ Key Vault        │ Standard, 10K ops       │ $0.30        │   │
│ │ Azure Monitor    │ 10 GB logs              │ $27.60       │   │
│ │ Bandwidth        │ 100 GB outbound         │ $8.07        │   │
│ │ ────────────────────────────────────────────────────────  │   │
│ │ TOTAL                                      │ $496.27/mo   │   │
│ │                                                            │   │
│ │ With 1yr RI:                                $380/mo       │   │
│ │ With 3yr RI + Hybrid Benefit:               $250/mo       │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Export] [Share] [Save]                                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### TCO Calculator (Total Cost of Ownership)

```
URL: azure.microsoft.com/pricing/tco/calculator/

Purpose: Compare on-premises costs vs Azure costs

Input:
├── Your current servers (type, count, specs)
├── Storage (TB, type)
├── Networking (bandwidth, VPNs)
├── IT labor costs
├── Software licenses
├── Data center costs (power, cooling, space)
└── Disaster recovery costs

Output:
├── Current on-premises cost: $XXX,XXX/year
├── Estimated Azure cost: $XX,XXX/year
├── Savings over 3 years: $XXX,XXX
└── Visual comparison chart

💡 Great for convincing management to migrate to Azure!
💡 Include ALL costs: hardware refresh, power, cooling, IT staff
```

---

## Part 11: Cost Optimization Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                COST OPTIMIZATION STRATEGIES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. AZURE HYBRID BENEFIT (if you have Windows/SQL licenses)          │
│  ├── Enable on VMs: Portal → VM → Configuration → Hybrid Benefit  │
│  ├── Enable on Azure SQL: During creation or after                  │
│  ├── Stack with RIs for maximum savings (up to 85%)                │
│  └── Audit: Which VMs could benefit? → Advisor recommends          │
│                                                                       │
│  2. RIGHT-SIZING                                                     │
│  ├── Azure Advisor → Cost → Right-size recommendations             │
│  ├── Check CPU/memory in Azure Monitor (if <30%, downsize)        │
│  ├── Move to B-series (burstable) for low-usage VMs               │
│  ├── Try Dv5/Ev5 (latest gen — better price/perf)                 │
│  ├── Consider ARM-based Dps/Eps (Ampere — 20% cheaper)            │
│  └── 💡 Right-size BEFORE buying RIs                                │
│                                                                       │
│  3. RESERVATIONS & SAVINGS PLANS                                     │
│  ├── Advisor shows RI recommendations based on usage               │
│  ├── Start with 1-year (less risk)                                  │
│  ├── Cover baseline (70-80%), use PAYG for peaks                   │
│  ├── Use instance size flexibility (same family covers multiple)   │
│  ├── Consider Savings Plans for multi-service flexibility          │
│  └── Review and adjust quarterly                                    │
│                                                                       │
│  4. SPOT VMs                                                         │
│  ├── CI/CD build agents → Spot (80% savings)                       │
│  ├── Batch processing → Spot VMSS                                  │
│  ├── AKS: Spot node pools for non-critical workloads               │
│  ├── Set eviction policy: Deallocate (to restart later)            │
│  └── Use multiple sizes for better availability                    │
│                                                                       │
│  5. SCHEDULING                                                       │
│  ├── Auto-shutdown on dev/staging VMs                               │
│  │   VM → Operations → Auto-shutdown → Set time (e.g., 20:00 IST)│
│  │   → Built-in feature! No extra setup needed.                   │
│  ├── Start VMs via Automation Account runbook (8:00 AM)            │
│  ├── Save ~65% on dev/staging compute costs                        │
│  ├── Azure DevTest Labs for dev environments                       │
│  │   → Auto-shutdown, auto-start, cost tracking                   │
│  └── AKS: Scale dev cluster to 0 nodes at night                   │
│                                                                       │
│  6. SERVERLESS & PaaS                                                │
│  ├── Azure SQL Serverless: Auto-pause (dev DBs → $0 when idle)    │
│  ├── Azure Functions (Consumption): Scale to zero                  │
│  ├── App Service: Multiple apps on one plan (shared cost)          │
│  ├── Cosmos DB Serverless: Pay per request (low-traffic apps)      │
│  ├── Azure Container Apps: Scale to zero                           │
│  └── AKS is FREE (control plane) — cheapest managed K8s!          │
│                                                                       │
│  7. STORAGE OPTIMIZATION                                             │
│  ├── Lifecycle management: Hot → Cool (30d) → Cold (90d) →        │
│  │   Archive (180d) → Delete (365d)                                 │
│  ├── Enable last access time tracking                              │
│  ├── Delete unattached managed disks                               │
│  ├── Use Standard HDD for non-critical data (vs Premium SSD)      │
│  ├── Delete old snapshots and images                               │
│  └── Consider Azure Files for shared storage (vs multiple disks)  │
│                                                                       │
│  8. NETWORKING                                                       │
│  ├── Azure Front Door / CDN for static content (cheaper egress)    │
│  ├── Private Endpoints instead of public access (no egress cost)   │
│  ├── NAT Gateway: Minimize or remove where possible               │
│  ├── Standard LB is paid ($18/month), Basic LB is free            │
│  ├── Use Service Endpoints for Azure-to-Azure traffic             │
│  └── Keep traffic in same region (intra-region is free)            │
│                                                                       │
│  9. MONITORING COSTS                                                 │
│  ├── Azure Monitor Logs: $2.76/GB — can be expensive!              │
│  │   → Set daily cap on workspace                                  │
│  │   → Use basic logs tier for verbose data (cheaper)              │
│  │   → Reduce log retention (default 30 days)                      │
│  │   → Filter out verbose diagnostics                              │
│  ├── Commitment tiers: 100 GB/day → $1.80/GB (35% cheaper)       │
│  └── Use Azure Monitor Agent (newer, more efficient)               │
│                                                                       │
│  10. CLEANUP WASTE                                                   │
│  ├── Unattached managed disks                                      │
│  ├── Unused public IPs ($3.65/month each)                          │
│  ├── Old snapshots and images                                      │
│  ├── Empty resource groups (no cost, but clutter)                  │
│  ├── Unused App Service Plans (even empty plan costs!)             │
│  ├── Idle Azure SQL databases                                      │
│  └── Azure Advisor catches most of these — check weekly!           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Real-World Patterns

### Startup ($400-1,500/month)

```
Cost management:
├── Free account + $200 credit to start
├── Use free tier services where possible
│   ├── AKS (free control plane!)
│   ├── Azure Functions (1M free requests)
│   ├── Azure SQL (250 GB free for 12 months)
│   └── B1S VM (750 hrs free for 12 months)
├── One budget with alerts at 50%, 80%, 100%
├── Auto-shutdown on all dev VMs
├── B-series VMs for low-usage services
├── Azure SQL Serverless for dev (auto-pause)
├── Advisor cost recommendations (free — check monthly)
└── Tags: Environment, Team, Application

Monthly cost example:
├── VMs (2x B2s): $60
├── Azure SQL (S1): $30
├── App Service (B1): $13
├── Blob Storage: $3
├── Networking: $10
├── Other: $10
└── Total: ~$126/month
```

### Mid-Size ($3,000-15,000/month)

```
Cost management:
├── MCA or EA agreement
├── Cost analysis dashboards per team
├── Budgets per subscription + per team (via tags)
├── Tags enforced via Azure Policy:
│   ├── Environment, Team, Application, CostCenter (mandatory)
│   ├── "Inherit tag from RG" policy (auto-propagate)
│   └── Weekly tag compliance report
├── Reservations:
│   ├── 1-year RIs for 70% of steady-state VMs
│   ├── Azure SQL RIs for production databases
│   └── Review every 6 months
├── Azure Hybrid Benefit on all Windows VMs + SQL
├── Auto-shutdown + auto-start for dev/staging
├── Spot VMs for CI/CD (Azure DevOps agents)
├── Azure SQL Serverless for dev/staging databases
├── Storage lifecycle policies on all storage accounts
├── Weekly Advisor review (15 min)
└── Quarterly optimization sprint

Monthly cost example:
├── VMs (10x D2s v5): $1,400 (after RIs + AHB)
├── AKS (8 nodes): $480 (free control plane!)
├── Azure SQL (3 instances): $380
├── App Service (2 plans): $200
├── Storage + CDN: $120
├── Networking: $150
├── Monitor + Logs: $80
├── Other: $190
└── Total: ~$3,000/month (saved $1,800 with optimization)
```

### Enterprise ($30,000+/month)

```
Cost management:
├── Enterprise Agreement with Azure Prepayment
├── Dedicated FinOps person/team
├── Cost Management API → Power BI executive dashboards
├── Chargeback model per business unit via cost allocation
├── Reservations + Savings Plans:
│   ├── 3-year RIs for stable production workloads (65% off)
│   ├── 1-year Savings Plans for growth workloads
│   ├── Spot for batch, ML training, CI/CD
│   ├── Coverage target: 70-80% of steady state
│   └── Quarterly review with finance
├── Azure Hybrid Benefit:
│   ├── All eligible Windows VMs → AHB enabled
│   ├── All SQL Server workloads → AHB enabled
│   ├── Combined with RIs: 80-85% savings on Windows/SQL
│   └── Track license utilization
├── Automated optimization:
│   ├── Azure Automation: Auto-stop/start schedules
│   ├── Tag compliance enforcement (Policy + remediation)
│   ├── Right-sizing automation (Advisor API + Automation)
│   └── Anomaly alerts via Cost Management
├── Advanced governance:
│   ├── Azure Policy: Deny oversized VMs in non-prod
│   ├── Azure Policy: Require cost-center tag
│   ├── Spending limits on sandbox subscriptions
│   └── Budget actions (alert when exceeded)
├── Cost export → Storage Account → Power BI
└── Monthly FinOps review with BU leaders

Typical savings:
├── RIs: 40-65% on committed compute
├── Azure Hybrid Benefit: 40-55% on Windows/SQL
├── RI + AHB: 80-85% on Windows VMs
├── Spot: 60-80% on batch/CI-CD
├── Right-sizing: 15-20%
├── Scheduling: 65% on non-prod
├── Total: 45-60% reduction from unoptimized bill
```

---

## Quick Reference: Console Navigation

| Task | Portal Path |
|------|------------|
| View current costs | Cost Management → Cost analysis |
| Create budget | Cost Management → Budgets → Add |
| View Advisor recs | Advisor → Cost |
| Buy reservations | Reservations → Add |
| Buy Savings Plan | Savings Plans → Add |
| View invoices | Cost Management + Billing → Invoices |
| Enable Hybrid Benefit | VM → Configuration → Azure Hybrid Benefit |
| Auto-shutdown VM | VM → Operations → Auto-shutdown |
| Pricing calculator | azure.microsoft.com/pricing/calculator |
| TCO calculator | azure.microsoft.com/pricing/tco/calculator |
| Export cost data | Cost Management → Exports |
| Tag compliance | Policy → Compliance |

---

## What's Next?

In the next chapter, we'll cover Azure Virtual Network (VNet) — networking fundamentals, subnets, NSGs, route tables, VNet peering, VPN Gateway, and private endpoints.

→ Next: [Chapter 6: Virtual Network (VNet)](06-virtual-networks.md)

---

*Last Updated: May 2026*
