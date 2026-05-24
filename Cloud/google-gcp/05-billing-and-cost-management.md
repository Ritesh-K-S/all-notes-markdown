# Chapter 5: Billing & Cost Management (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: GCP Pricing Fundamentals](#part-1-gcp-pricing-fundamentals)
- [Part 2: Pricing Models](#part-2-pricing-models)
- [Part 3: Free Tier & Always Free](#part-3-free-tier--always-free)
- [Part 4: Billing Accounts & Hierarchy](#part-4-billing-accounts--hierarchy)
- [Part 5: Billing Console Walkthrough](#part-5-billing-console-walkthrough)
- [Part 6: Budgets & Alerts](#part-6-budgets--alerts)
- [Part 7: Billing Export to BigQuery](#part-7-billing-export-to-bigquery)
- [Part 8: Cost Optimization Strategies](#part-8-cost-optimization-strategies)
- [Part 9: GCP Pricing Calculator](#part-9-gcp-pricing-calculator)

---

## Overview

GCP billing is tied to Billing Accounts, which are linked to projects. Understanding GCP's pricing, committed use discounts, and billing tools is essential for controlling cloud spend.

```
What you'll learn:
├── GCP pricing fundamentals
├── Pricing models (On-Demand, Sustained Use, CUDs, Preemptible/Spot)
├── Free Tier & Always Free products
├── Billing Accounts & hierarchy
├── Billing Console walkthrough
├── Budgets & Alerts
├── Billing Export to BigQuery (power analytics)
├── Labels for cost allocation
├── Committed Use Discounts (CUDs) deep dive
├── Preemptible / Spot VMs
├── Cost optimization strategies
├── GCP Pricing Calculator
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: GCP Pricing Fundamentals

### How GCP Charges You

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GCP PRICING PRINCIPLES                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. PAY-AS-YOU-GO (per second billing for most services)             │
│     └── VM runs for 42 minutes 15 seconds → pay for exactly that   │
│     └── Minimum: 1 minute, then per-second                         │
│                                                                       │
│  2. SUSTAINED USE DISCOUNTS (automatic, no commitment!)              │
│     └── Use a VM for 25%+ of the month → auto-discount up to 30%   │
│     └── No action needed — completely automatic                     │
│     └── ⚡ Unique to GCP — AWS/Azure don't have this               │
│                                                                       │
│  3. COMMITTED USE DISCOUNTS (1 or 3 year commitment)                 │
│     └── Commit to vCPUs + memory → up to 57% discount              │
│                                                                       │
│  4. CUSTOMER-FRIENDLY PRICING                                        │
│     └── No charge for many operations that AWS charges for          │
│     └── Sub-dollar billing (no minimums for many services)          │
│                                                                       │
│  5. DATA TRANSFER                                                    │
│     └── Ingress (data IN): FREE (almost always)                    │
│     └── Egress (data OUT to internet):                              │
│         ├── First 200 GB/month: FREE (as of Aug 2024)              │
│         ├── 1-10 TB: $0.12/GB                                      │
│         └── 10+ TB: tiered pricing (decreasing)                     │
│     └── Egress between zones (same region): $0.01/GB               │
│     └── Egress between regions: $0.01-0.15/GB (varies)             │
│     └── Egress within same zone: FREE                               │
│                                                                       │
│  Billing cycle: Monthly (1st to last day)                            │
│  Payment: Credit card, bank transfer, invoice (for eligible)         │
│  Currency: USD (can set local currency for display)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### What Costs Money

```
┌──────────────────────────────────────────────────────────────────────┐
│ Service           │ What You Pay For                                  │
├──────────────────────────────────────────────────────────────────────┤
│ Compute Engine    │ Per second (after 1 min minimum), vCPU type,      │
│ (VMs)             │ memory, region. Stopped = no compute but disk cost│
│                   │ Sustained use discounts auto-applied              │
│                   │                                                    │
│ Persistent Disk   │ Per GB-month provisioned, snapshots extra         │
│                   │ Standard: ~$0.04/GB, SSD: ~$0.17/GB              │
│                   │                                                    │
│ Cloud Storage     │ Per GB-month stored + operations + egress         │
│ (GCS)             │ Standard: $0.020/GB, Nearline: $0.010/GB         │
│                   │ Coldline: $0.004/GB, Archive: $0.0012/GB         │
│                   │                                                    │
│ Cloud SQL         │ Per vCPU-hour + per GB-RAM-hour + storage +       │
│                   │ backups beyond 7 days. HA = 2x compute cost      │
│                   │                                                    │
│ Cloud Run         │ Per vCPU-second + per GB-memory-second +          │
│                   │ per request. Scale to zero = $0 when idle!       │
│                   │ Free tier: 2M requests/month                     │
│                   │                                                    │
│ Cloud Functions   │ Per invocation + compute time (GB-seconds)        │
│                   │ Free: 2M invocations/month                       │
│                   │                                                    │
│ GKE               │ Standard: $0.10/hr per cluster ($73/month)       │
│                   │ Autopilot: Per pod vCPU + memory (no cluster fee)│
│                   │ + node costs                                     │
│                   │                                                    │
│ BigQuery          │ On-demand: $6.25/TB queried (first 1TB free)     │
│                   │ Storage: $0.02/GB (first 10 GB free)             │
│                   │ Flat-rate: Fixed slots (predictable pricing)      │
│                   │                                                    │
│ Load Balancer     │ Per rule/hour + per GB processed                 │
│                   │ First 5 rules included (~$18/month base)         │
│                   │                                                    │
│ Cloud NAT         │ Per NAT gateway/hour + per GB processed          │
│                   │ ~$0.044/hr + $0.045/GB                           │
│                   │                                                    │
│ Cloud DNS         │ $0.20/zone/month + $0.40/million queries         │
│                   │                                                    │
│ Pub/Sub           │ First 10 GB/month free, then $40/TiB             │
│                   │                                                    │
│ Secret Manager    │ 6 versions free, then $0.06/version/month        │
│                   │ + $0.03/10K access operations                    │
│                   │                                                    │
│ Cloud Logging     │ First 50 GB/month free, then $0.50/GB            │
│                   │ ⚠️ Can get expensive with verbose logging!       │
│                   │                                                    │
│ Artifact Registry │ Per GB stored + egress                           │
│                   │ First 500 MB free                                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Pricing Models

### Sustained Use Discounts (Automatic!)

```
┌─────────────────────────────────────────────────────────────────────┐
│               SUSTAINED USE DISCOUNTS (SUDs)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How it works: The longer you run a VM in a month, the cheaper        │
│ it gets. No commitment or action needed — fully automatic!           │
│                                                                       │
│ Discount tiers (% of month used → discount):                        │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │  Usage        │ % of Month │ Effective Discount              │    │
│ │ ─────────────────────────────────────────────────────────── │    │
│ │  0-25%        │ 0-25%      │ 0% (full price)                │    │
│ │  25-50%       │ 25-50%     │ 20% discount on this portion   │    │
│ │  50-75%       │ 50-75%     │ 40% discount on this portion   │    │
│ │  75-100%      │ 75-100%    │ 60% discount on this portion   │    │
│ │                                                              │    │
│ │  Full month (100%): ~30% effective discount overall          │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Applies to:                                                          │
│ ├── Compute Engine VMs (N1, N2, N2D, E2 families)                  │
│ ├── GKE nodes (same VMs underneath)                                 │
│ ├── Cloud SQL instances                                             │
│ └── ⚠️ NOT: E2 shared-core, Spot VMs, sole-tenant nodes            │
│                                                                       │
│ Aggregation: GCP aggregates usage across same machine type           │
│ in same region. Two n2-standard-4 VMs each running 50% of month    │
│ = 100% usage = maximum discount!                                     │
│                                                                       │
│ ⚡ This is FREE MONEY — no action required!                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Committed Use Discounts (CUDs)

```
┌─────────────────────────────────────────────────────────────────────┐
│              COMMITTED USE DISCOUNTS (CUDs)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Commit to a minimum amount of vCPUs and memory for             │
│       1 or 3 years in exchange for a discount.                       │
│                                                                       │
│ Types:                                                               │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ 1. Resource-based CUDs (Compute Engine)                      │    │
│ │    ├── Commit to: vCPUs + memory (per region)               │    │
│ │    ├── Flexible: Applies to any machine type in the region  │    │
│ │    ├── 1 year: Up to 37% discount                           │    │
│ │    ├── 3 years: Up to 55% discount                          │    │
│ │    └── Covers: Compute Engine, GKE, Dataproc                │    │
│ │                                                              │    │
│ │ 2. Spend-based CUDs (specific services)                     │    │
│ │    ├── Commit to: Minimum $/hr spend                        │    │
│ │    ├── Available for: Cloud SQL, BigQuery, Cloud Run, etc.  │    │
│ │    ├── 1 year: 25% discount                                 │    │
│ │    ├── 3 years: 52% discount                                │    │
│ │    └── More flexible than resource-based                    │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ CUD vs AWS comparison:                                               │
│ ┌──────────────────┬──────────────────┬────────────────────────┐    │
│ │ Feature          │ GCP CUD          │ AWS RI/Savings Plan    │    │
│ │ ─────────────── │ ──────────────── │ ────────────────────── │    │
│ │ Commitment       │ vCPUs + RAM      │ Instance type or $/hr  │    │
│ │ Flexibility      │ Any VM in region │ Varies by plan type    │    │
│ │ Auto-discount    │ Sustained Use    │ None                   │    │
│ │ Payment          │ Monthly only     │ Upfront/Partial/Monthly│    │
│ │ Cancellation     │ Cannot cancel    │ Cannot cancel          │    │
│ └──────────────────┴──────────────────┴────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Purchasing CUDs

```
Console → Compute Engine → Committed use discounts → Purchase

┌─────────────────────────────────────────────────────────────────┐
│            PURCHASE COMMITTED USE DISCOUNT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:         [prod-commitment-2026]                             │
│ Region:       [asia-south1 ▼]                                   │
│ Commitment type: ○ General purpose  ○ Memory optimized          │
│                  ○ Compute optimized ○ Accelerator optimized    │
│                                                                   │
│ Duration:     ○ 1 year  ○ 3 years                               │
│                                                                   │
│ Resources:                                                       │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ vCPUs:    [20] (number of vCPUs to commit)                │   │
│ │ Memory:   [80 GB] (amount of RAM to commit)               │   │
│ │                                                            │   │
│ │ Monthly cost:                                              │   │
│ │   Without CUD:  $XXX/month (On-Demand)                    │   │
│ │   With CUD:     $XXX/month (after discount)               │   │
│ │   Savings:       $XXX/month (XX%)                          │   │
│ │                                                            │   │
│ │ 💡 These vCPUs/RAM auto-apply to any eligible VMs         │   │
│ │    running in asia-south1                                  │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Purchase]                                                       │
│ ⚠️ Cannot cancel! You pay even if you don't use the resources   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  gcloud compute commitments create prod-commitment-2026 \
    --region=asia-south1 \
    --plan=36-month \
    --resources=vcpu=20,memory=80GB \
    --type=GENERAL_PURPOSE
```

### Preemptible & Spot VMs

```
┌─────────────────────────────────────────────────────────────────────┐
│             PREEMPTIBLE / SPOT VMs                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ SPOT VMs (replacement for Preemptible, same concept):                │
│ ├── Up to 60-91% discount vs On-Demand                              │
│ ├── Can be terminated at ANY time (30-second warning)               │
│ ├── No guaranteed availability                                       │
│ ├── No live migration during maintenance                            │
│ ├── No auto-restart (unlike regular VMs)                            │
│ ├── No SLA                                                          │
│                                                                       │
│ Difference from Preemptible:                                         │
│ ├── Preemptible: Max 24-hour lifetime, always terminated at 24h    │
│ ├── Spot: No max lifetime (but can be preempted anytime)           │
│ ├── Spot is the newer, preferred option                             │
│ └── Both have same pricing                                          │
│                                                                       │
│ Best for:                                                            │
│ ├── Batch processing jobs                                           │
│ ├── CI/CD build agents                                              │
│ ├── Data processing / ETL                                           │
│ ├── ML model training (with checkpointing)                          │
│ ├── Rendering / media processing                                    │
│ └── Stateless web servers (behind LB, with MIG)                    │
│                                                                       │
│ NOT for:                                                             │
│ ├── Databases                                                       │
│ ├── Single-instance applications                                    │
│ ├── Stateful services without proper handling                       │
│ └── Production workloads requiring SLA                              │
│                                                                       │
│ Create Spot VM:                                                      │
│ Console → Compute Engine → Create instance                           │
│ → Availability policies → VM provisioning model: Spot              │
│                                                                       │
│ CLI:                                                                 │
│ gcloud compute instances create batch-worker-01 \                    │
│   --provisioning-model=SPOT \                                        │
│   --instance-termination-action=STOP \                               │
│   --machine-type=n2-standard-4 \                                     │
│   --zone=asia-south1-a                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Free Tier & Always Free

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GCP FREE TIER                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. FREE TRIAL ($300 credit for 90 days — new accounts)              │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── $300 credit to spend on any GCP service                  │    │
│ │ ├── Valid for 90 days from signup                            │    │
│ │ ├── Full access to all services                              │    │
│ │ ├── Won't auto-charge after trial (must upgrade manually)   │    │
│ │ ├── ⚠️ Some restrictions: Max 8 vCPUs at once, no GPUs      │    │
│ │ └── After trial: Resources stopped, must upgrade to continue │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 2. ALWAYS FREE (never expires, no credit needed)                     │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Compute Engine:     1 e2-micro VM/month (us-central1 only!) │    │
│ │                     30 GB standard PD, 1 GB egress           │    │
│ │                                                               │    │
│ │ Cloud Storage:      5 GB regional (us-central1, us-east1,   │    │
│ │                     us-west1 only!)                          │    │
│ │                                                               │    │
│ │ BigQuery:           1 TB queries/month + 10 GB storage       │    │
│ │                                                               │    │
│ │ Cloud Functions:    2M invocations/month, 400K GB-seconds    │    │
│ │                                                               │    │
│ │ Cloud Run:          2M requests/month, 360K vCPU-seconds     │    │
│ │                                                               │    │
│ │ Cloud Build:        120 build-minutes/day                    │    │
│ │                                                               │    │
│ │ Artifact Registry:  500 MB storage                           │    │
│ │                                                               │    │
│ │ Cloud Logging:      50 GB/month                              │    │
│ │                                                               │    │
│ │ Pub/Sub:            10 GB/month                              │    │
│ │                                                               │    │
│ │ Cloud Shell:        Free interactive shell (5 GB home dir)   │    │
│ │                                                               │    │
│ │ Vision AI:          1,000 units/month                        │    │
│ │ Speech-to-Text:     60 minutes/month                        │    │
│ │ Natural Language:   5,000 units/month                       │    │
│ │ Translation:        500K characters/month                   │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ ⚠️ COMMON FREE TIER TRAPS:                                          │
│ ├── Free VM only in specific US regions (not asia-south1!)          │
│ ├── Free storage only in specific US regions                        │
│ ├── Running a VM 24/7 in wrong region = charges                    │
│ ├── Cloud Logging over 50 GB (easy to exceed with verbose apps)    │
│ ├── Egress exceeding 200 GB free tier                               │
│ └── Forgetting resources after free trial expires                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Billing Accounts & Hierarchy

### How Billing Works in GCP

```
┌─────────────────────────────────────────────────────────────────────┐
│                GCP BILLING HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Organization (techcorp.com)                                          │
│ │                                                                    │
│ ├── Billing Account: "TechCorp Main Billing"                       │
│ │   ├── Payment method: Credit card / Invoice                      │
│ │   ├── Contact: finance@techcorp.com                               │
│ │   ├── Linked projects:                                            │
│ │   │   ├── tc-prod-backend (Production)                            │
│ │   │   ├── tc-prod-frontend                                        │
│ │   │   ├── tc-staging-all                                          │
│ │   │   └── tc-shared-services                                      │
│ │   └── Cost = sum of all linked projects                           │
│ │                                                                    │
│ └── Billing Account: "TechCorp Dev Billing"                        │
│     ├── Separate payment / separate budget                          │
│     └── Linked projects:                                            │
│         ├── tc-dev-backend                                          │
│         ├── tc-dev-frontend                                         │
│         └── tc-sandbox                                              │
│                                                                       │
│ KEY RULES:                                                           │
│ ├── Each project links to exactly ONE billing account               │
│ ├── A billing account can have MANY projects                        │
│ ├── Billing account is separate from resource hierarchy             │
│ ├── You can have multiple billing accounts per org                  │
│ ├── Disabling billing on a project → all resources stopped          │
│ └── Billing admins don't need project access (separation)           │
│                                                                       │
│ Billing IAM Roles:                                                   │
│ ├── Billing Account Administrator: Full control of billing          │
│ ├── Billing Account User: Link projects to billing account         │
│ ├── Billing Account Viewer: View costs and transactions            │
│ ├── Project Billing Manager: Link/unlink project to billing        │
│ └── Billing Account Costs Manager: Manage budgets, exports         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Billing Console Walkthrough

```
Console → Billing (left nav or navigation menu)

┌─────────────────────────────────────────────────────────────────┐
│                   BILLING CONSOLE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Left navigation:                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Overview            ← Dashboard with spend summary        │   │
│ │ Reports             ← Detailed cost graphs & analysis     │   │
│ │ Cost table          ← Tabular cost breakdown              │   │
│ │ Cost breakdown      ← By project, service, SKU            │   │
│ │ Budgets & alerts    ← Set spending limits                  │   │
│ │ Committed use       ← CUD management                      │   │
│ │   discounts                                                │   │
│ │ Pricing             ← View pricing for services            │   │
│ │ Transactions        ← Payment history                      │   │
│ │ Credits             ← View available credits               │   │
│ │ Billing export      ← Export to BigQuery / CSV             │   │
│ │ Account management  ← Billing account settings             │   │
│ │ Payment settings    ← Payment methods, invoices            │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Overview shows:                                                  │
│ ├── This month's cost: $2,340.50                                │
│ ├── Forecasted month-end: $3,200.00                             │
│ ├── Last month total: $3,100.00                                 │
│ ├── Trend (month-over-month)                                    │
│ ├── Cost by project (pie chart)                                 │
│ └── Promotional credits remaining                               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Billing Reports

```
Billing → Reports

┌─────────────────────────────────────────────────────────────────┐
│                    BILLING REPORTS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Filters:                                                         │
│ ├── Time range: [Last 3 months ▼]                               │
│ ├── Group by: [Project ▼] (Project/Service/SKU/Location/Label) │
│ ├── Projects: [All / Specific]                                  │
│ ├── Services: [All / Specific]                                  │
│ ├── Locations: [All / Specific region]                          │
│ ├── Labels: [env=production / team=backend]                     │
│ └── Credits: [Include / Exclude]                                │
│                                                                   │
│ Report view:                                                     │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │  $4000 ┤                                                   │   │
│ │        │              ████                                 │   │
│ │  $3000 ┤         ████ ████                                 │   │
│ │        │    ████ ████ ████                                  │   │
│ │  $2000 ┤    ████ ████ ████                                 │   │
│ │        │    ████ ████ ████                                  │   │
│ │  $1000 ┤    ████ ████ ████                                 │   │
│ │        │    ████ ████ ████                                  │   │
│ │     $0 └────┬────┬────┬────                                │   │
│ │           Mar  Apr  May                                    │   │
│ │                                                            │   │
│ │  ████ Compute Engine  ████ Cloud SQL  ████ Cloud Storage   │   │
│ │  ████ Networking      ████ Other                           │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Table view (toggle):                                             │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Service          │ This Month │ Last Month │ Change       │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ Compute Engine   │ $1,240     │ $1,150     │ +$90 (↑8%)  │   │
│ │ Cloud SQL        │ $580       │ $580       │ $0 (→)      │   │
│ │ Cloud Storage    │ $120       │ $110       │ +$10 (↑9%)  │   │
│ │ Networking       │ $230       │ $200       │ +$30 (↑15%) │   │
│ │ Cloud Run        │ $45        │ $40        │ +$5 (↑12%)  │   │
│ │ Other            │ $125       │ $120       │ +$5 (↑4%)   │   │
│ │ ─────────────────────────────────────────────────────────  │   │
│ │ Total            │ $2,340     │ $2,200     │ +$140 (↑6%) │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Budgets & Alerts

### Creating a Budget

```
Billing → Budgets & alerts → Create budget

┌─────────────────────────────────────────────────────────────────┐
│                    CREATE BUDGET                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Scope                                                    │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Name:           [Production Monthly Budget]                │   │
│ │ Time range:     [Monthly ▼] (Monthly/Quarterly/Annual/    │   │
│ │                  Custom)                                   │   │
│ │                                                            │   │
│ │ Projects:       [tc-prod-backend ▼]                       │   │
│ │                 (select specific or all)                   │   │
│ │                                                            │   │
│ │ Services:       [All services ▼]                           │   │
│ │                 (or filter to specific services)           │   │
│ │                                                            │   │
│ │ Labels:         [env: production ▼]                        │   │
│ │                 (filter by label for cost tracking)        │   │
│ │                                                            │   │
│ │ Credits:        ☑ Include credits in cost calculation      │   │
│ │ Discounts:      ☑ Include discounts                        │   │
│ │ Promotions:     ☑ Include promotions                       │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 2: Amount                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Budget type:                                               │   │
│ │ ○ Specified amount: [$3,500]                              │   │
│ │ ○ Last month's spend (auto-adjusting)                     │   │
│ │ ○ Last period's spend                                     │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 3: Actions (alert thresholds)                               │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Threshold rules:                                           │   │
│ │ ┌─────────┬──────────┬────────────────────────────────┐   │   │
│ │ │ % of    │ Trigger  │ Action                          │   │   │
│ │ │ budget  │ type     │                                 │   │   │
│ │ ├─────────┼──────────┼────────────────────────────────┤   │   │
│ │ │ 50%     │ Actual   │ Email billing admins            │   │   │
│ │ │ 80%     │ Actual   │ Email + notify Pub/Sub          │   │   │
│ │ │ 90%     │ Forecasted│ Email + notify Pub/Sub         │   │   │
│ │ │ 100%    │ Actual   │ Email + notify Pub/Sub          │   │   │
│ │ └─────────┴──────────┴────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ Email notifications to:                                    │   │
│ │ ☑ Billing admins and users (default)                      │   │
│ │ ☑ Monitoring email notification channels:                  │   │
│ │   [devops@techcorp.com, finance@techcorp.com]             │   │
│ │                                                            │   │
│ │ Connect a Pub/Sub topic:                                   │   │
│ │ [projects/tc-prod-backend/topics/budget-alerts]           │   │
│ │ → Use for automated responses (Cloud Function triggered)  │   │
│ │ → Example: Auto-stop non-critical VMs when 100% reached  │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Finish]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Production Monthly Budget" \
    --budget-amount=3500 \
    --filter-projects="projects/tc-prod-backend" \
    --threshold-rule=percent=0.5,basis=current-spend \
    --threshold-rule=percent=0.8,basis=current-spend \
    --threshold-rule=percent=1.0,basis=current-spend \
    --all-updates-rule-pubsub-topic="projects/tc-prod-backend/topics/budget-alerts"
```

### Automating Budget Responses

```
Budget → Pub/Sub → Cloud Function → Take action

Example: Auto-disable billing when budget exceeded (nuclear option):

Cloud Function (Python):
  import base64, json
  from googleapiclient import discovery

  def budget_alert(data, context):
      pubsub_data = json.loads(base64.b64decode(data['data']))
      cost = pubsub_data['costAmount']
      budget = pubsub_data['budgetAmount']
      
      if cost >= budget:
          # Disable billing on project (stops all resources!)
          billing = discovery.build('cloudbilling', 'v1')
          billing.projects().updateBillingInfo(
              name='projects/tc-sandbox',
              body={'billingAccountName': ''}
          ).execute()
          print(f"Billing disabled! Cost: ${cost}, Budget: ${budget}")

⚠️ Only use for sandbox/dev! Disabling billing = all resources stop!
For production: Send alerts only, let humans decide.
```

---

## Part 7: Billing Export to BigQuery

### The Power Analysis Tool

```
Billing → Billing export → BigQuery export

┌─────────────────────────────────────────────────────────────────┐
│              BILLING EXPORT TO BIGQUERY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Export types:                                                     │
│ ├── Standard usage cost: Daily cost data (most common)          │
│ ├── Detailed usage cost: Hourly, resource-level detail          │
│ │   (much more data, higher BigQuery storage cost)              │
│ └── Pricing: Export pricing data for all SKUs                    │
│                                                                   │
│ Setup:                                                           │
│ 1. Create BigQuery dataset: billing_export                      │
│ 2. Configure export (Billing → Billing export)                  │
│ 3. Wait 24-48 hours for data to appear                          │
│ 4. Query with SQL!                                               │
│                                                                   │
│ ⚠️ This is the MOST POWERFUL cost analysis tool in GCP           │
│    Far more flexible than the console UI                        │
│    Free to export! (Pay only for BigQuery queries + storage)    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Useful queries:

-- Total cost by project (this month)
SELECT project.id, SUM(cost) as total_cost
FROM `billing_export.gcp_billing_export_v1_XXXXXX`
WHERE invoice.month = '202605'
GROUP BY project.id
ORDER BY total_cost DESC;

-- Cost by service
SELECT service.description, SUM(cost) as total_cost
FROM `billing_export.gcp_billing_export_v1_XXXXXX`
WHERE invoice.month = '202605'
GROUP BY service.description
ORDER BY total_cost DESC;

-- Cost by label (team)
SELECT 
  (SELECT value FROM UNNEST(labels) WHERE key = 'team') as team,
  SUM(cost) as total_cost
FROM `billing_export.gcp_billing_export_v1_XXXXXX`
WHERE invoice.month = '202605'
GROUP BY team
ORDER BY total_cost DESC;

-- Daily cost trend (last 30 days)
SELECT DATE(usage_start_time) as date, SUM(cost) as daily_cost
FROM `billing_export.gcp_billing_export_v1_XXXXXX`
WHERE usage_start_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY date
ORDER BY date;

-- Top 10 most expensive resources
SELECT project.id, service.description, sku.description,
       SUM(cost) as total_cost
FROM `billing_export.gcp_billing_export_v1_XXXXXX`
WHERE invoice.month = '202605'
GROUP BY 1, 2, 3
ORDER BY total_cost DESC
LIMIT 10;

-- Unlabeled resource costs
SELECT project.id, service.description, SUM(cost) as total_cost
FROM `billing_export.gcp_billing_export_v1_XXXXXX`
WHERE invoice.month = '202605'
  AND NOT EXISTS (SELECT 1 FROM UNNEST(labels) WHERE key = 'team')
GROUP BY 1, 2
ORDER BY total_cost DESC;
```

---

## Part 8: Cost Optimization Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                COST OPTIMIZATION STRATEGIES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. RIGHT-SIZING                                                     │
│  ├── Use Recommender Hub → VM right-sizing recommendations         │
│  ├── Console → Compute Engine → VM instances → Recommendations     │
│  ├── Check CPU/memory in Cloud Monitoring (if <30%, downsize)      │
│  ├── Try E2 instances (cost-optimized, good for general workloads) │
│  ├── Consider Tau T2D (best price/perf for scale-out)              │
│  └── 💡 Right-size BEFORE buying CUDs                               │
│                                                                       │
│  2. SPOT VMs (for fault-tolerant workloads)                          │
│  ├── CI/CD pipelines → Spot (60-91% savings)                       │
│  ├── Batch processing → Spot MIG with auto-healing                 │
│  ├── GKE: Spot node pools for non-critical workloads               │
│  ├── Dataproc: Spot VMs for Spark/Hadoop workers                   │
│  └── Use Managed Instance Groups for auto-replacement              │
│                                                                       │
│  3. COMMITTED USE DISCOUNTS                                          │
│  ├── Analyze 3+ months of usage                                    │
│  ├── Commit to 70-80% of steady-state baseline                     │
│  ├── 1-year first (less risk), then 3-year for proven workloads    │
│  ├── Resource-based CUDs for compute (most flexible)               │
│  └── Spend-based CUDs for Cloud SQL, BigQuery                      │
│                                                                       │
│  4. SERVERLESS (pay per use, scale to zero)                          │
│  ├── Cloud Run instead of always-on VMs for APIs                   │
│  ├── Cloud Functions for event-driven processing                   │
│  ├── BigQuery on-demand instead of provisioned compute             │
│  ├── Firestore/Datastore instead of always-on Cloud SQL            │
│  └── Cloud Scheduler + Cloud Functions for cron jobs               │
│                                                                       │
│  5. STORAGE OPTIMIZATION                                             │
│  ├── Lifecycle rules: Standard → Nearline (30d) → Coldline (90d)  │
│  ├── Object Lifecycle Management (auto-transition, auto-delete)    │
│  ├── Autoclass (auto-moves between storage classes)                │
│  │   → Set and forget! GCP handles transitions automatically      │
│  ├── Delete old snapshots and unused disks                         │
│  └── Persistent Disk: Use pd-balanced instead of pd-ssd if OK     │
│                                                                       │
│  6. NETWORKING                                                       │
│  ├── Cloud NAT: Expensive! Minimize outbound traffic               │
│  ├── Use Standard Tier networking (cheaper, single region egress)  │
│  ├── Use Private Google Access (no egress for Google API calls)    │
│  ├── Cloud CDN for static content (cheaper than raw egress)        │
│  └── Keep traffic in same zone when possible (free intra-zone)     │
│                                                                       │
│  7. SCHEDULING                                                       │
│  ├── Stop dev/staging VMs nights & weekends                        │
│  │   Instance schedule (built-in!) or Cloud Scheduler + Function   │
│  ├── Save ~65% on non-production VM costs                          │
│  ├── GKE: Scale dev cluster to 0 nodes at night                    │
│  └── Cloud SQL: Stop dev instances (restart takes ~1 min)          │
│                                                                       │
│  8. LOGGING & MONITORING                                             │
│  ├── Cloud Logging: 50 GB free, then $0.50/GB                     │
│  │   → Exclude verbose logs (debug-level) from ingestion           │
│  │   → Use log exclusion filters                                   │
│  │   → Route to cheaper storage (GCS) for archival                 │
│  ├── Set log retention to minimum needed (default 30 days)         │
│  └── Use log sampling for high-volume services                     │
│                                                                       │
│  9. CLEANUP WASTE                                                    │
│  ├── Unattached persistent disks                                   │
│  ├── Unused static external IPs ($0.01/hr ≈ $7.30/month)          │
│  ├── Old snapshots                                                  │
│  ├── Idle GKE clusters ($73/month per cluster just for control)    │
│  ├── Orphaned load balancers                                       │
│  ├── Unused Cloud SQL instances                                    │
│  └── Use Active Assist Recommender (automated recommendations)     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: GCP Pricing Calculator

```
URL: cloud.google.com/products/calculator

┌─────────────────────────────────────────────────────────────────┐
│               GCP PRICING CALCULATOR                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Add services → configure → get estimate                         │
│                                                                   │
│ Example estimate for a web application:                          │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Service          │ Config                  │ Monthly Cost  │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ Compute (2x)     │ e2-standard-2, On-Demand│ $98.00       │   │
│ │ Persistent Disk  │ 100 GB pd-balanced      │ $10.00       │   │
│ │ Cloud SQL        │ db-custom-2-8192, HA    │ $185.00      │   │
│ │ Cloud SQL storage│ 50 GB SSD               │ $8.50        │   │
│ │ Cloud Storage    │ 100 GB Standard         │ $2.00        │   │
│ │ Cloud Run        │ 2M requests, 1 vCPU     │ $40.00       │   │
│ │ Cloud NAT        │ 1 gateway + 50 GB       │ $34.50       │   │
│ │ Cloud DNS        │ 1 zone + 1M queries     │ $0.60        │   │
│ │ Load Balancer    │ 5 rules, 100 GB data    │ $22.00       │   │
│ │ Cloud Logging    │ 20 GB (within free)     │ $0.00        │   │
│ │ ────────────────────────────────────────────────────────  │   │
│ │ TOTAL                                      │ $400.60/mo   │   │
│ │                                                            │   │
│ │ With SUDs (full month): ~$370/mo                          │   │
│ │ With 1-yr CUDs:         ~$310/mo                          │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Share estimate] [Download as PDF/CSV]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Real-World Patterns

### Startup ($300-1,500/month)

```
Cost management:
├── Start with $300 free trial credit
├── Use Always Free tier where possible
├── Single budget with alerts at 50%, 80%, 100%
├── Billing export to BigQuery (free to set up)
├── Use e2-micro/small for non-prod
├── Cloud Run for APIs (scale to zero = $0 when idle)
├── Spot VMs for CI/CD
├── Label everything (env, team, app)
├── Monthly manual bill review
└── Consider Standard Tier networking (cheaper)

Monthly cost example:
├── Compute (2 VMs): $98
├── Cloud SQL (small): $50
├── Cloud Storage: $5
├── Cloud Run (APIs): $20
├── Networking: $15
├── Other: $12
└── Total: ~$200/month
```

### Mid-Size ($3,000-15,000/month)

```
Cost management:
├── Billing export to BigQuery + Looker Studio dashboard
├── Budgets per project + per team (via labels)
├── Labels enforced: env, team, app, cost-center
├── CUDs:
│   ├── 1-year resource-based for 70% of baseline compute
│   ├── Spend-based CUD for Cloud SQL
│   └── Review every 6 months
├── Spot VMs for CI/CD and batch processing
├── Instance scheduling for dev/staging (65% savings)
├── Cloud Storage Autoclass (auto-tiering)
├── Right-sizing via Recommender (monthly review)
├── Cloud Run for variable-traffic APIs
├── Weekly cost review (15 min automated report)
└── Quarterly optimization sprint

Monthly cost example:
├── Compute (10 VMs): $1,800 (after CUDs + SUDs)
├── Cloud SQL (2 instances): $450
├── GKE: $300 (Autopilot)
├── Cloud Storage: $100
├── Cloud Run: $80
├── Networking: $180
├── Logging/Monitoring: $90
└── Total: ~$3,000/month (saved $1,200 with optimizations)
```

### Enterprise ($30,000+/month)

```
Cost management:
├── FinOps practice with dedicated person
├── Multiple billing accounts (prod vs dev)
├── Billing export → BigQuery → Looker Studio executive dashboard
├── Custom BigQuery views for chargeback per BU
├── CUDs:
│   ├── 3-year resource-based for stable workloads (55% savings)
│   ├── 1-year for growth workloads
│   ├── Spend-based for Cloud SQL + BigQuery
│   └── CUD coverage target: 70-80%
├── Negotiated pricing (for large spend):
│   ├── Contact Google Cloud sales for custom pricing
│   ├── Enterprise agreements available
│   └── Sustained commitment discounts
├── Automated optimization:
│   ├── Recommender API → auto-apply right-sizing
│   ├── Spot node pools in GKE for non-critical
│   ├── Autoclass for all Cloud Storage buckets
│   └── Label compliance enforcement (Cloud Function)
├── Advanced analytics:
│   ├── Resource-level cost attribution
│   ├── Unit economics (cost per transaction/user)
│   ├── Efficiency metrics per team
│   └── Anomaly detection via BigQuery ML
└── Monthly FinOps review with BU leaders

Typical savings achieved:
├── CUDs: 35-55% on committed compute
├── SUDs: ~30% automatic (full month VMs)
├── Spot: 60-80% on batch/CI-CD
├── Right-sizing: 15-25%
├── Scheduling: 65% on non-prod
├── Storage tiering: 40-60% on old data
└── Total: 40-55% reduction from unoptimized
```

---

## Quick Reference: Console Navigation

| Task | Path |
|------|------|
| View current costs | Billing → Overview |
| Detailed cost analysis | Billing → Reports |
| Create budget | Billing → Budgets & alerts → Create |
| Buy CUDs | Compute Engine → Committed use discounts |
| Billing export | Billing → Billing export |
| View credits | Billing → Credits |
| Price calculator | cloud.google.com/products/calculator |
| Right-sizing recs | Compute Engine → VM instances → Recommendations |
| Recommender Hub | Recommender Hub (service) |
| Cost by label | Billing → Reports → Group by Label |

---

## What's Next?

In the next chapter, we'll cover VPC (Virtual Private Cloud) — networking fundamentals, subnets, firewall rules, Cloud NAT, VPN, Shared VPC, and VPC peering.

→ Next: [Chapter 6: VPC - Virtual Private Cloud](06-vpc.md)

---

*Last Updated: May 2026*
