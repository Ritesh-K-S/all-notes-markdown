# Chapter 5: Billing & Cost Management (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AWS Pricing Fundamentals](#part-1-aws-pricing-fundamentals)
- [Part 2: Pricing Models](#part-2-pricing-models)
- [Part 3: AWS Free Tier](#part-3-aws-free-tier)
- [Part 4: AWS Billing Console Walkthrough](#part-4-aws-billing-console-walkthrough)
- [Part 5: Cost Explorer](#part-5-cost-explorer)
- [Part 6: Budgets & Alerts](#part-6-budgets--alerts)
- [Part 7: Cost Allocation Tags](#part-7-cost-allocation-tags)
- [Part 8: Consolidated Billing (AWS Organizations)](#part-8-consolidated-billing-aws-organizations)
- [Part 9: Savings Plans Deep Dive](#part-9-savings-plans-deep-dive)

---

## Overview

### Why Should Billing Be Your First AWS Skill?

> **Real-World Analogy:** AWS billing is like a utility bill — electricity, water, and gas combined. You pay for compute (electricity), storage (water), and data transfer (gas), each metered differently. But unlike home utilities, cloud bills can go from $50 to $5,000 overnight if you accidentally leave the "lights on" — like running 20 GPU instances you forgot about.

Horror stories of $50,000 surprise AWS bills are not myths — they happen to beginners regularly. The **#1 beginner task** after creating an account is setting up billing alerts. Understanding pricing models also directly impacts architecture decisions — choosing Spot over On-Demand can save your company 70%.

Understanding AWS pricing and cost management is critical — cloud bills can spiral fast without proper controls. This chapter covers every pricing model, billing tools, budgets, alerts, and cost optimization strategies.

```
What you'll learn:
├── AWS pricing fundamentals (how cloud billing works)
├── Pricing models (On-Demand, Reserved, Savings Plans, Spot)
├── Free Tier (what's included, traps to avoid)
├── AWS Billing Console (walkthrough of every section)
├── Cost Explorer (analyzing spending)
├── Budgets & Alerts (never get surprised)
├── Cost Allocation Tags (track by team/app)
├── AWS Organizations billing (consolidated billing)
├── Savings Plans & Reserved Instances (deep dive)
├── Spot Instances (up to 90% discount)
├── Cost optimization strategies
├── AWS Pricing Calculator
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: AWS Pricing Fundamentals

### How AWS Charges You

```
┌─────────────────────────────────────────────────────────────────────┐
│                   AWS PRICING PRINCIPLES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. PAY-AS-YOU-GO (most services)                                    │
│     └── Use a VM for 3 hours → pay for 3 hours                     │
│                                                                       │
│  2. PAY PER USE (consumption-based)                                  │
│     └── Lambda: pay per request + compute time                      │
│     └── S3: pay per GB stored + requests                            │
│                                                                       │
│  3. PAY LESS WHEN YOU RESERVE (commitment discounts)                 │
│     └── Commit for 1 or 3 years → up to 72% discount               │
│                                                                       │
│  4. PAY LESS AS YOU USE MORE (volume discounts)                      │
│     └── S3: First 50 TB = $0.023/GB, next 450 TB = $0.022/GB       │
│                                                                       │
│  5. NO COST FOR DATA IN (mostly)                                     │
│     └── Data transfer IN to AWS = free                              │
│     └── Data transfer OUT = charged (per GB)                        │
│     └── Data transfer between AZs = charged ($0.01/GB)              │
│     └── Data transfer within same AZ = free (using private IP)      │
│                                                                       │
│  Billing cycle: Monthly (1st to last day of month)                   │
│  Payment: Credit card or invoice (enterprise)                        │
│  Currency: USD (can set other currencies for display)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### What Costs Money (Common Services)

```
┌──────────────────────────────────────────────────────────────────────┐
│ Service         │ What You Pay For                                    │
├──────────────────────────────────────────────────────────────────────┤
│ EC2             │ Per hour/second (depends on OS), instance type,     │
│                 │ region. Stopped VM = no compute charge but EBS cost │
│                 │                                                      │
│ EBS (disks)     │ Per GB-month provisioned + IOPS (for io1/io2)      │
│                 │ Snapshots: per GB-month stored                      │
│                 │                                                      │
│ S3              │ Per GB-month stored + requests (GET/PUT) +          │
│                 │ data transfer out. Different rates per storage class│
│                 │                                                      │
│ RDS             │ Per hour (instance) + storage GB + backup beyond    │
│                 │ DB size + data transfer. Multi-AZ = 2x instance    │
│                 │                                                      │
│ Lambda          │ Per request ($0.20/million) + compute time (GB-sec) │
│                 │ First 1M requests/month free                        │
│                 │                                                      │
│ ALB/NLB         │ Per hour + LCU/NLCU (load balancer capacity units) │
│                 │                                                      │
│ NAT Gateway     │ Per hour ($0.045) + per GB processed ($0.045)      │
│                 │ ⚠️ One of the sneakiest costs!                      │
│                 │                                                      │
│ CloudFront      │ Per GB data out + per 10K requests                 │
│                 │ Often CHEAPER than direct data transfer out!        │
│                 │                                                      │
│ Data Transfer   │ IN = free, OUT to internet = $0.09/GB (first 10TB)│
│                 │ Between regions = $0.02/GB, between AZs = $0.01/GB│
│                 │                                                      │
│ Route 53        │ $0.50/hosted zone/month + $0.40/million queries    │
│                 │                                                      │
│ Elastic IP      │ FREE when attached to running instance             │
│                 │ ⚠️ $0.005/hour when NOT attached (waste charge)    │
│                 │                                                      │
│ Secrets Manager │ $0.40/secret/month + $0.05/10K API calls          │
│                 │                                                      │
│ CloudWatch      │ 10 custom metrics free, then $0.30/metric/month   │
│                 │ Logs: $0.50/GB ingested + $0.03/GB stored          │
│                 │                                                      │
│ EKS             │ $0.10/hour per cluster ($73/month just for control)│
│                 │ + node costs (EC2 or Fargate)                       │
│                 │                                                      │
│ DynamoDB        │ On-demand: per read/write unit                     │
│                 │ Provisioned: per RCU/WCU per hour                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Pricing Models

### The Four Pricing Models

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PRICING MODELS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. ON-DEMAND (full price, no commitment)                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Pay by the second (Linux) or hour (Windows)              │   │
│  │ ├── No upfront cost, no long-term commitment                 │   │
│  │ ├── Most expensive per-unit price                            │   │
│  │ ├── Best for: Unpredictable workloads, testing, short-term   │   │
│  │ │                                                             │   │
│  │ │ Example: m5.large in ap-south-1                            │   │
│  │ │ Price: ~$0.092/hour ≈ $67/month                            │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  2. RESERVED INSTANCES (1 or 3 year commitment)                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Commit to specific instance type + region                │   │
│  │ ├── Up to 72% discount vs On-Demand                          │   │
│  │ ├── Payment options:                                         │   │
│  │ │   ├── All Upfront (biggest discount)                       │   │
│  │ │   ├── Partial Upfront (balanced)                           │   │
│  │ │   └── No Upfront (smallest discount, monthly payments)     │   │
│  │ ├── Types:                                                   │   │
│  │ │   ├── Standard RI: Fixed instance type (bigger discount)   │   │
│  │ │   └── Convertible RI: Can change instance type (smaller)   │   │
│  │ ├── Best for: Steady-state, predictable workloads            │   │
│  │ │                                                             │   │
│  │ │ Example: m5.large, 1yr, All Upfront                        │   │
│  │ │ Price: ~$0.057/hour ≈ $41/month (38% savings)              │   │
│  │ │                                                             │   │
│  │ │ Example: m5.large, 3yr, All Upfront                        │   │
│  │ │ Price: ~$0.036/hour ≈ $26/month (61% savings)              │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  3. SAVINGS PLANS (flexible commitment — RECOMMENDED)                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Commit to $/hour spend (not instance type)               │   │
│  │ ├── Up to 72% discount (same as RI)                          │   │
│  │ ├── More flexible than Reserved Instances                    │   │
│  │ ├── Types:                                                   │   │
│  │ │   ├── Compute SP: Any instance family, region, OS, tenancy│   │
│  │ │   │   (Most flexible — covers EC2, Fargate, Lambda)       │   │
│  │ │   └── EC2 Instance SP: Specific instance family + region  │   │
│  │ │       (Bigger discount, less flexible)                     │   │
│  │ ├── Best for: Flexible workloads across instance families    │   │
│  │ │                                                             │   │
│  │ │ Example: Commit $10/hour for 1 year                        │   │
│  │ │ → All compute up to $10/hour gets discounted rate          │   │
│  │ │ → Usage above $10/hour charged at On-Demand               │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  4. SPOT INSTANCES (up to 90% discount, can be interrupted)          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ├── Bid for unused AWS capacity                              │   │
│  │ ├── Up to 90% discount vs On-Demand                          │   │
│  │ ├── ⚠️ Can be terminated with 2-minute warning               │   │
│  │ ├── Best for: Fault-tolerant, flexible workloads             │   │
│  │ │   ├── Batch processing, data analysis                      │   │
│  │ │   ├── CI/CD build agents                                   │   │
│  │ │   ├── Stateless web servers (behind LB)                    │   │
│  │ │   └── Big data / ML training jobs                          │   │
│  │ ├── NOT for: Databases, single-instance apps, stateful work  │   │
│  │ │                                                             │   │
│  │ │ Example: m5.large Spot in ap-south-1                       │   │
│  │ │ Price: ~$0.028/hour ≈ $20/month (70% savings)              │   │
│  │ └──                                                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  COMPARISON:                                                         │
│  ┌────────────────┬─────────┬─────────┬──────────┬──────────────┐  │
│  │ Model          │ Discount│ Commit  │ Flexible │ Interruptible│  │
│  ├────────────────┼─────────┼─────────┼──────────┼──────────────┤  │
│  │ On-Demand      │ 0%      │ None    │ Full     │ No           │  │
│  │ Reserved (1yr) │ 30-40%  │ 1 year  │ Low      │ No           │  │
│  │ Reserved (3yr) │ 50-72%  │ 3 years │ Low      │ No           │  │
│  │ Savings Plan   │ 30-72%  │ 1-3 yr  │ High     │ No           │  │
│  │ Spot           │ 60-90%  │ None    │ Full     │ YES          │  │
│  └────────────────┴─────────┴─────────┴──────────┴──────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: AWS Free Tier

### Free Tier Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS FREE TIER                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Three types of Free Tier:                                            │
│                                                                       │
│ 1. 12-MONTH FREE (starts when you create account)                   │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ EC2:        750 hours/month of t2.micro (or t3.micro)        │    │
│ │ S3:         5 GB storage + 20K GET + 2K PUT                  │    │
│ │ RDS:        750 hours/month of db.t2.micro (single-AZ)       │    │
│ │ CloudFront: 1 TB data transfer out/month                     │    │
│ │ ELB:        750 hours/month of Classic or ALB                │    │
│ │ EBS:        30 GB of General Purpose (gp2/gp3)              │    │
│ │ SNS:        1M publishes                                     │    │
│ │ SQS:        1M requests                                      │    │
│ │ ElastiCache:750 hours of cache.t2.micro                     │    │
│ │                                                               │    │
│ │ ⚠️ After 12 months, these start charging On-Demand!          │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 2. ALWAYS FREE (never expires)                                       │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Lambda:     1M requests + 400,000 GB-seconds/month           │    │
│ │ DynamoDB:   25 GB storage + 25 RCU + 25 WCU                 │    │
│ │ CloudWatch: 10 custom metrics + 10 alarms                   │    │
│ │ SNS:        1M mobile push                                   │    │
│ │ SES:        62,000 outbound emails/month (from EC2)         │    │
│ │ CodeBuild:  100 build minutes/month                          │    │
│ │ CodeCommit: 5 active users                                   │    │
│ │ X-Ray:      100,000 traces/month                             │    │
│ │ Glacier:    10 GB retrievals/month                           │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 3. TRIALS (specific services, limited time)                          │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Amazon Inspector:    90 days free                            │    │
│ │ Amazon GuardDuty:    30 days free                            │    │
│ │ Amazon Macie:        30 days free                            │    │
│ │ AWS Config:          30 days free (rules evaluations)        │    │
│ │ Amazon SageMaker:    2 months free (specific instances)      │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ ⚠️ COMMON FREE TIER TRAPS:                                          │
│ ├── Running multiple t2.micro → 750 hrs shared across ALL           │
│ ├── Launching in wrong region (some free tiers are region-specific) │
│ ├── Forgetting to delete resources after testing                    │
│ ├── EBS volumes left after terminating EC2                          │
│ ├── Elastic IPs not attached to running instances                   │
│ ├── NAT Gateway (NOT free tier)                                     │
│ └── Data transfer out exceeding free limit                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: AWS Billing Console Walkthrough

### Billing Dashboard

```
Portal → Billing and Cost Management (top-right menu → Billing)

┌─────────────────────────────────────────────────────────────────┐
│                  BILLING DASHBOARD                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Left navigation:                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Home                    ← Dashboard overview               │   │
│ │ Bills                   ← Detailed monthly bill            │   │
│ │ Cost Explorer           ← Analyze spending (graphs)        │   │
│ │ Budgets                 ← Set budget limits + alerts        │   │
│ │ Cost & Usage Reports    ← Detailed CSV exports             │   │
│ │ Cost Anomaly Detection  ← AI-detected unusual spending     │   │
│ │ Savings Plans           ← Purchase/manage commitments      │   │
│ │ Reservations            ← Reserved Instances               │   │
│ │ Credits                 ← AWS credits balance              │   │
│ │ Purchase Orders         ← PO management                    │   │
│ │ Cost Categories         ← Group costs logically            │   │
│ │ Cost Allocation Tags    ← Tags for cost tracking           │   │
│ │ Free Tier               ← Track free tier usage            │   │
│ │ Preferences             ← Invoice, PDF, settings           │   │
│ │ Tax Settings            ← GST/VAT registration             │   │
│ │ Payment Methods         ← Credit card, bank account        │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Dashboard shows:                                                 │
│ ├── Month-to-date spend: $2,847.23                              │
│ ├── Forecasted month-end: $4,150.00                             │
│ ├── Last month: $3,920.15                                       │
│ ├── Top 5 services by cost                                      │
│ └── Free tier usage summary                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Bills Page

```
Billing → Bills

┌─────────────────────────────────────────────────────────────────┐
│                     MONTHLY BILL                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Date range: [May 2026 ▼]                                        │
│                                                                   │
│ Summary:                                                         │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Total charges:                          $3,920.15          │   │
│ │   AWS service charges:                  $3,820.15          │   │
│ │   Tax (GST 18%):                        $687.63            │   │
│ │   Credits applied:                     -$587.63            │   │
│ │   ─────────────────────────────────────────────           │   │
│ │   Total:                                $3,920.15          │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Charges by service:                                              │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ▼ Amazon Elastic Compute Cloud         $1,240.50           │   │
│ │     Asia Pacific (Mumbai)              $1,140.50           │   │
│ │       m5.large - On-Demand             $680.00             │   │
│ │       t3.medium - On-Demand            $320.50             │   │
│ │       EBS Volume (gp3, 500GB)          $140.00             │   │
│ │     US East (N. Virginia)              $100.00             │   │
│ │                                                            │   │
│ │ ▼ Amazon RDS                           $890.00             │   │
│ │     db.r5.large Multi-AZ              $760.00              │   │
│ │     Storage (100GB)                    $130.00              │   │
│ │                                                            │   │
│ │ ▼ Amazon S3                            $245.30             │   │
│ │ ▼ Amazon CloudFront                    $156.20             │   │
│ │ ▼ AWS Lambda                           $12.40              │   │
│ │ ▼ Data Transfer                        $340.00             │   │
│ │ ▼ Other services...                    $935.75             │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Download CSV] [Download PDF Invoice]                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Cost Explorer

### Using Cost Explorer

```
Billing → Cost Explorer

┌─────────────────────────────────────────────────────────────────┐
│                   COST EXPLORER                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Time range: [Last 6 months ▼]                                   │
│ Granularity: [Monthly ▼] (Daily / Monthly / Hourly)             │
│ Group by: [Service ▼]                                           │
│                                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │                                                            │   │
│ │  $5000 ┤                                                   │   │
│ │        │         ████                                      │   │
│ │  $4000 ┤    ████ ████ ████                                 │   │
│ │        │    ████ ████ ████ ████                             │   │
│ │  $3000 ┤    ████ ████ ████ ████ ████                       │   │
│ │        │    ████ ████ ████ ████ ████                        │   │
│ │  $2000 ┤████████ ████ ████ ████ ████                       │   │
│ │        │████████ ████ ████ ████ ████                        │   │
│ │  $1000 ┤████████ ████ ████ ████ ████                       │   │
│ │        │████████ ████ ████ ████ ████                        │   │
│ │     $0 └────┬────┬────┬────┬────┬────                      │   │
│ │           Dec  Jan  Feb  Mar  Apr  May                     │   │
│ │                                                            │   │
│ │  ████ EC2  ████ RDS  ████ S3  ████ Other                   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Filters:                                                         │
│ ├── Service: [All / Specific services]                          │
│ ├── Linked account: [Specific account in Organization]          │
│ ├── Region: [ap-south-1 / us-east-1 / All]                     │
│ ├── Instance type: [m5.large / t3.medium / All]                 │
│ ├── Tag: [Environment=production]                               │
│ ├── Usage type: [specific usage]                                │
│ └── Purchase option: [On-Demand / Reserved / Spot / All]        │
│                                                                   │
│ Group by options:                                                │
│ ├── Service (most common)                                       │
│ ├── Linked Account                                              │
│ ├── Region                                                      │
│ ├── Instance Type                                               │
│ ├── Usage Type                                                  │
│ ├── Tag (e.g., group by Environment tag)                        │
│ ├── Purchase Option                                             │
│ ├── API Operation                                               │
│ └── Cost Category                                               │
│                                                                   │
│ Save as report: [Save] → Can create multiple saved views        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Key Cost Explorer analyses:
├── "Which service costs the most?" → Group by Service
├── "How much does production cost?" → Filter by tag Environment=production
├── "Which team spends the most?" → Group by tag Team
├── "Are we saving with RIs?" → Filter by Purchase Option
├── "Cost by region?" → Group by Region
└── "Month-over-month trend?" → Monthly granularity, 12 months
```

---

## Part 6: Budgets & Alerts

### Creating a Budget

```
Billing → Budgets → Create budget

┌─────────────────────────────────────────────────────────────────┐
│                    CREATE BUDGET                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Budget type:                                                     │
│ ├── ○ Cost budget (track spending in $)                         │
│ ├── ○ Usage budget (track specific usage, e.g., EC2 hours)      │
│ ├── ○ Savings Plans budget                                      │
│ └── ○ Reservation budget                                        │
│                                                                   │
│ Selected: Cost budget                                            │
│                                                                   │
│ Budget details:                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Name:           [Monthly-Total-Budget]                     │   │
│ │ Period:         [Monthly ▼] (Daily/Monthly/Quarterly/Annual│   │
│ │ Budget amount:  ○ Fixed: [$5,000]                          │   │
│ │                 ○ Auto-adjusting (based on historical)      │   │
│ │ Start month:    [May 2026]                                 │   │
│ │ End month:      [None — recurring]                         │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Scope (filters — optional):                                      │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Filter by service:    [All services]                       │   │
│ │ Filter by account:    [All accounts]                       │   │
│ │ Filter by tag:        [Environment: production]            │   │
│ │ Filter by region:     [All regions]                        │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Alert thresholds:                                                │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Alert 1:                                                   │   │
│ │   Threshold: [80] % of budgeted amount                    │   │
│ │   Trigger:   ○ Actual (when spend hits 80%)               │   │
│ │              ○ Forecasted (when forecast hits 80%)        │   │
│ │   Notify:    [devops@techcorp.com, finance@techcorp.com]  │   │
│ │              [SNS Topic ARN] (for automation)              │   │
│ │                                                            │   │
│ │ Alert 2:                                                   │   │
│ │   Threshold: [100] % of budgeted amount                   │   │
│ │   Trigger:   Actual                                       │   │
│ │   Notify:    [cto@techcorp.com]                            │   │
│ │                                                            │   │
│ │ Alert 3:                                                   │   │
│ │   Threshold: [100] % of budgeted amount                   │   │
│ │   Trigger:   Forecasted                                   │   │
│ │   Notify:    [finance@techcorp.com]                        │   │
│ │                                                            │   │
│ │ [+ Add another threshold]                                  │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Budget actions (optional — auto-respond):                        │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ When threshold exceeded:                                   │   │
│ │ ├── Apply IAM policy (deny new resource creation)         │   │
│ │ ├── Stop EC2/RDS instances                                │   │
│ │ └── Apply SCP (block specific actions)                    │   │
│ │                                                            │   │
│ │ ⚠️ Be careful with auto-actions in production!             │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create budget]                                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  aws budgets create-budget \
    --account-id 123456789012 \
    --budget '{
      "BudgetName": "Monthly-Total",
      "BudgetLimit": {"Amount": "5000", "Unit": "USD"},
      "TimeUnit": "MONTHLY",
      "BudgetType": "COST"
    }' \
    --notifications-with-subscribers '[{
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80
      },
      "Subscribers": [{
        "SubscriptionType": "EMAIL",
        "Address": "devops@techcorp.com"
      }]
    }]'
```

### Recommended Budget Structure

```
For a mid-size company:

Budget 1: Total monthly spend
  Amount: $5,000
  Alerts: 50%, 80%, 100% (actual + forecasted)
  Notify: Finance + CTO

Budget 2: Production spend
  Amount: $3,500
  Filter: Tag Environment=production
  Alerts: 80%, 100%
  Notify: DevOps lead

Budget 3: Development spend
  Amount: $1,000
  Filter: Tag Environment=development
  Alerts: 80%, 100%
  Action: Apply SCP to deny new resources at 120%

Budget 4: Per-team budgets
  Amount: $1,500 (backend team)
  Filter: Tag Team=backend
  Alerts: 80%, 100%
  Notify: Team lead

Budget 5: Unused resources alert
  Type: Usage
  Track: EC2 Running Hours
  Compare to previous month (auto-adjusting)
```

---

## Part 7: Cost Allocation Tags

### Setting Up Cost Allocation Tags

```
Billing → Cost Allocation Tags

┌─────────────────────────────────────────────────────────────────┐
│                COST ALLOCATION TAGS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Two types:                                                       │
│                                                                   │
│ 1. AWS-Generated Tags (auto-created by AWS)                     │
│    ├── aws:createdBy (who created the resource)                 │
│    ├── aws:cloudformation:stack-name (CloudFormation)            │
│    └── Must be activated in Billing console                     │
│                                                                   │
│ 2. User-Defined Tags (your custom tags)                         │
│    ├── You tag resources with key-value pairs                   │
│    ├── Must activate each tag key in Billing console            │
│    ├── Takes up to 24 hours to appear in billing                │
│    └── Only ACTIVATED tags show in Cost Explorer                │
│                                                                   │
│ Activation:                                                      │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ User-defined cost allocation tags:                         │   │
│ │                                                            │   │
│ │ Tag Key         │ Status      │ Action                     │   │
│ │ ─────────────── │ ─────────── │ ──────                     │   │
│ │ Environment     │ ✅ Active    │ [Deactivate]              │   │
│ │ Team            │ ✅ Active    │ [Deactivate]              │   │
│ │ Application     │ ✅ Active    │ [Deactivate]              │   │
│ │ CostCenter      │ ⬚ Inactive  │ [Activate]               │   │
│ │ Owner           │ ⬚ Inactive  │ [Activate]               │   │
│ │                                                            │   │
│ │ [Activate selected] [Deactivate selected]                  │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ ⚠️ Tags must exist on resources AND be activated here            │
│    to show up in Cost Explorer and billing reports               │
│                                                                   │
│ ⚠️ Activation takes up to 24 hours                               │
│ ⚠️ Tags only affect billing data AFTER activation (not retroactive)│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Consolidated Billing (AWS Organizations)

```
┌─────────────────────────────────────────────────────────────────────┐
│                 CONSOLIDATED BILLING                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When using AWS Organizations, all accounts get ONE bill:             │
│                                                                       │
│ Management Account (payer)                                           │
│ ├── Receives single bill for all accounts                           │
│ ├── Can see cost breakdown per account                              │
│ ├── Volume discounts aggregated across all accounts                 │
│ └── RI/Savings Plan benefits shared across accounts                 │
│                                                                       │
│ Benefits:                                                            │
│ ├── Volume pricing: All accounts' usage combined                    │
│ │   Account A: 40 TB S3 + Account B: 20 TB S3 = 60 TB total       │
│ │   → Both get pricing for 60 TB tier (cheaper per GB!)            │
│ │                                                                   │
│ ├── RI sharing: RI bought in Account A can apply to Account B      │
│ │   (unless disabled)                                               │
│ │                                                                   │
│ ├── Savings Plan sharing: Same as RI sharing                        │
│ │                                                                   │
│ └── Single payment method for all accounts                          │
│                                                                       │
│ Disable RI/SP sharing (if you want per-account billing):            │
│ Organizations → Settings → Disable credit sharing                  │
│                                                                       │
│ Per-account cost visibility:                                         │
│ Cost Explorer → Group by: Linked Account                            │
│ → Shows how much each account spends                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Savings Plans Deep Dive

### Purchasing a Savings Plan

```
Billing → Savings Plans → Purchase Savings Plans

┌─────────────────────────────────────────────────────────────────┐
│               PURCHASE SAVINGS PLAN                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Recommendations (based on your usage):                           │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Based on last 30 days of usage, AWS recommends:            │   │
│ │                                                            │   │
│ │ Compute Savings Plan:                                      │   │
│ │   Commitment: $8.50/hour (1 year, No Upfront)             │   │
│ │   Estimated monthly savings: $1,240                        │   │
│ │   Savings vs On-Demand: 32%                               │   │
│ │                                                            │   │
│ │ EC2 Instance Savings Plan:                                 │   │
│ │   Commitment: $6.20/hour (1 year, All Upfront)            │   │
│ │   Instance family: m5, Region: ap-south-1                 │   │
│ │   Estimated monthly savings: $1,580                        │   │
│ │   Savings vs On-Demand: 38%                               │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Configure:                                                       │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Savings Plan type:                                         │   │
│ │   ○ Compute (any instance family/region/OS/tenancy)       │   │
│ │   ○ EC2 Instance (specific family + region)               │   │
│ │   ○ SageMaker                                             │   │
│ │                                                            │   │
│ │ Term: ○ 1 year  ○ 3 years                                │   │
│ │                                                            │   │
│ │ Hourly commitment: [$8.50]                                │   │
│ │                                                            │   │
│ │ Payment option:                                            │   │
│ │   ○ All Upfront (biggest discount)                        │   │
│ │   ○ Partial Upfront                                       │   │
│ │   ○ No Upfront (monthly payments)                         │   │
│ │                                                            │   │
│ │ Start date: [Today]                                       │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ How it works after purchase:                                     │
│ ├── Your compute usage up to $8.50/hr gets discounted rate      │
│ ├── Usage above $8.50/hr charged at On-Demand                  │
│ ├── Discount applied automatically (no tagging needed)          │
│ ├── Covers: EC2, Fargate, Lambda (Compute SP)                  │
│ └── Can add more Savings Plans if usage grows                   │
│                                                                   │
│ [Add to cart] → [Submit order]                                   │
│ ⚠️ Cannot cancel after purchase!                                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Cost Anomaly Detection

```
Billing → Cost Anomaly Detection

┌─────────────────────────────────────────────────────────────────┐
│               COST ANOMALY DETECTION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: AI/ML-powered detection of unusual spending patterns       │
│ Cost: Free!                                                      │
│                                                                   │
│ Create monitor:                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Monitor type:                                              │   │
│ │ ○ AWS Services (individual service anomalies)             │   │
│ │ ○ Linked Accounts (per-account anomalies)                 │   │
│ │ ○ Cost Categories (custom groupings)                      │   │
│ │ ○ Cost Allocation Tags (tag-based anomalies)              │   │
│ │                                                            │   │
│ │ Alert subscription:                                        │   │
│ │ Threshold: [$50] (minimum anomaly amount to alert)        │   │
│ │ Frequency: [Individual alerts / Daily summary / Weekly]   │   │
│ │ Recipients: [devops@techcorp.com]                          │   │
│ │ SNS Topic: [arn:aws:sns:...] (for automation)             │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Example alert:                                                   │
│ "Anomaly detected: EC2 spending increased by $340 (85%)         │
│  on May 15. Root cause: 12 new m5.xlarge instances launched     │
│  in us-east-1 by user john.doe@techcorp.com"                    │
│                                                                   │
│ ✅ Enable this on Day 1 — it's free and catches surprises!       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 11: AWS Pricing Calculator

```
URL: calculator.aws (https://calculator.aws)

┌─────────────────────────────────────────────────────────────────┐
│                AWS PRICING CALCULATOR                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Use BEFORE deploying to estimate monthly costs                   │
│                                                                   │
│ Example estimate for a web application:                          │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Service          │ Config                  │ Monthly Cost  │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ EC2 (2x)         │ m5.large, On-Demand     │ $134.00      │   │
│ │ ALB              │ 10 LCUs average         │ $22.50       │   │
│ │ RDS              │ db.r5.large, Multi-AZ   │ $380.00      │   │
│ │ S3               │ 100 GB, Standard        │ $2.30        │   │
│ │ CloudFront       │ 500 GB transfer         │ $42.50       │   │
│ │ NAT Gateway      │ 1 NAT, 100 GB processed │ $37.35       │   │
│ │ Route 53         │ 1 zone, 1M queries      │ $0.90        │   │
│ │ CloudWatch       │ 20 metrics, 5 GB logs   │ $8.50        │   │
│ │ Data Transfer    │ 200 GB out              │ $18.00       │   │
│ │ ────────────────────────────────────────────────────────  │   │
│ │ TOTAL                                      │ $646.05/mo   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Actions: [Share link] [Export as PDF] [Export as CSV]             │
│                                                                   │
│ 💡 Share the estimate URL with your team for review              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Cost Optimization Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                COST OPTIMIZATION STRATEGIES                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. RIGHT-SIZING (match resources to actual usage)                   │
│  ├── Use AWS Compute Optimizer (free recommendations)               │
│  ├── Check CPU/memory utilization in CloudWatch                     │
│  ├── Downsize over-provisioned instances                            │
│  │   m5.xlarge (4 CPU, 16 GB) at 15% CPU → try m5.large (2 CPU)   │
│  ├── Use Graviton (ARM) instances: 20% cheaper, 40% better perf   │
│  └── 💡 Right-size BEFORE buying Savings Plans/RIs                   │
│                                                                       │
│  2. SPOT INSTANCES (for fault-tolerant workloads)                    │
│  ├── CI/CD build agents → Spot (save 70%)                           │
│  ├── Batch processing → Spot Fleet                                  │
│  ├── Auto Scaling Group with Spot mixed (On-Demand base + Spot)    │
│  └── Use multiple instance types (diversify for availability)       │
│                                                                       │
│  3. SAVINGS PLANS / RESERVED INSTANCES                               │
│  ├── Analyze 30-day usage in Cost Explorer                          │
│  ├── Use AWS recommendations (Billing → Savings Plans)              │
│  ├── Start with Compute SP (most flexible)                          │
│  ├── 1-year first (less risk), then 3-year for stable loads        │
│  └── Cover your baseline, use On-Demand/Spot for peaks             │
│                                                                       │
│  4. STORAGE OPTIMIZATION                                             │
│  ├── S3 Lifecycle policies: Move old data to cheaper tiers          │
│  │   Standard → IA (30 days) → Glacier (90 days) → Deep Archive   │
│  ├── Use S3 Intelligent-Tiering (auto-moves data)                  │
│  ├── Delete unattached EBS volumes                                  │
│  ├── Delete old snapshots                                           │
│  ├── Enable S3 Object Lock instead of keeping copies               │
│  └── Use gp3 instead of gp2 EBS (20% cheaper, better IOPS)        │
│                                                                       │
│  5. NETWORKING                                                       │
│  ├── ⚠️ NAT Gateway: $0.045/hr + $0.045/GB — very expensive!       │
│  │   → Use VPC Endpoints for S3/DynamoDB (free, no NAT needed)     │
│  │   → Consolidate NAT Gateways if possible                        │
│  ├── Use CloudFront for data transfer (cheaper than direct out)    │
│  ├── Keep traffic in same AZ when possible ($0.01/GB cross-AZ)     │
│  └── Use PrivateLink instead of public internet paths              │
│                                                                       │
│  6. AUTO-SCALING & SCHEDULING                                        │
│  ├── Auto-scale down during off-hours                               │
│  ├── Schedule: Stop dev/staging instances nights & weekends         │
│  │   → Save ~65% on dev/staging EC2 costs                          │
│  ├── Use AWS Instance Scheduler (official solution)                 │
│  └── Lambda: Optimize memory (right-size for cost/perf balance)    │
│                                                                       │
│  7. DATABASE OPTIMIZATION                                            │
│  ├── Aurora Serverless for variable workloads (scale to zero)      │
│  ├── Use Reserved Instances for production RDS                     │
│  ├── Read replicas instead of bigger instance (scale reads)        │
│  ├── DynamoDB: On-Demand for unpredictable, Provisioned for steady │
│  └── Delete unused databases and snapshots                         │
│                                                                       │
│  8. CLEAN UP WASTE                                                   │
│  ├── Unattached EBS volumes                                        │
│  ├── Unused Elastic IPs ($3.60/month each!)                        │
│  ├── Old snapshots                                                  │
│  ├── Unused load balancers ($16/month minimum)                     │
│  ├── Idle NAT Gateways                                             │
│  ├── Stopped RDS instances (auto-restart after 7 days!)            │
│  └── Use AWS Trusted Advisor (free checks for some)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Real-World Patterns

### Startup ($500-2,000/month)

```
Cost management:
├── Enable Free Tier tracking alerts
├── One monthly budget ($2,000) with alerts at 50%, 80%, 100%
├── Cost Anomaly Detection (free — enable immediately)
├── Use t3.micro/small for non-prod (free tier eligible)
├── Use Spot for CI/CD (GitHub Actions self-hosted runners)
├── Single NAT Gateway (or NAT Instance for <$5/month)
├── gp3 volumes everywhere (cheaper than gp2)
├── S3 Intelligent-Tiering (set and forget)
└── Review bill manually every month (it's small enough)

Monthly cost breakdown example:
├── EC2 (2 instances): $140
├── RDS (db.t3.small): $50
├── S3 + CloudFront: $25
├── ALB: $23
├── NAT Gateway: $37
├── Other (Route53, CloudWatch, etc.): $25
└── Total: ~$300/month
```

### Mid-Size ($5,000-20,000/month)

```
Cost management:
├── AWS Organizations with consolidated billing
├── Cost allocation tags: Environment, Team, Application, CostCenter
├── Monthly budgets per team + total
├── Cost Anomaly Detection on all services
├── Savings Plans:
│   ├── Analyze 3 months of usage
│   ├── Compute SP for 70% of baseline ($8-10/hr)
│   ├── Keep 30% On-Demand for flexibility
│   └── Review quarterly
├── Right-sizing review monthly (Compute Optimizer)
├── Instance Scheduler for dev/staging (stop nights/weekends)
├── S3 lifecycle policies (30 → IA, 90 → Glacier)
├── VPC endpoints for S3 and DynamoDB (eliminate NAT costs)
├── Weekly cost review meeting (15 min)
└── Quarterly optimization sprint

Monthly cost breakdown example:
├── EC2 (15 instances): $2,800 (after Savings Plan)
├── RDS (3 instances): $1,200
├── EKS: $400 (control plane + nodes)
├── S3 + CloudFront: $350
├── Data Transfer: $500
├── NAT Gateway: $200
├── Other: $550
└── Total: ~$6,000/month (saved $2,400 with SP + optimization)
```

### Enterprise ($50,000+/month)

```
Cost management:
├── Dedicated FinOps team/person
├── AWS Organizations with 20+ accounts
├── Cost & Usage Reports → S3 → Athena/QuickSight dashboards
├── Cost Categories for business mapping
├── Chargeback/showback model per BU
├── Savings Plans + RIs:
│   ├── 3-year All Upfront for stable workloads (60%+ savings)
│   ├── 1-year Compute SP for growth workloads
│   ├── Spot Fleet for batch, ML training, CI/CD
│   └── RI coverage target: 70-80%
├── Enterprise Discount Program (EDP):
│   ├── Commit total annual spend → additional discount
│   └── Negotiate with AWS account manager
├── Automated optimization:
│   ├── Lambda: Auto-stop idle resources
│   ├── Auto-scaling with predictive scaling
│   ├── Right-sizing pipeline (monthly auto-recommendations)
│   └── Tag compliance enforcement (untagged → alert → fix)
├── FinOps tools: CloudHealth, Apptio, or native AWS
├── Monthly FinOps review with stakeholders
└── Annual cloud spend optimization audit

Typical savings achieved:
├── Savings Plans/RIs: 30-40% on compute
├── Spot: 70% on batch/CI-CD
├── Right-sizing: 15-20% on over-provisioned
├── Scheduling: 65% on non-prod
├── Storage tiering: 50% on old data
└── Total: 35-50% reduction from unoptimized bill
```

---

## Pricing Model Decision Tree

```
Which EC2 pricing model should I use?

Is your workload steady for 1+ years?
├── YES → Can you commit to specific instance type/family?
│         ├── YES → EC2 Instance Savings Plan (best discount, ~72%)
│         └── NO  → Compute Savings Plan (flexible, ~66%)
└── NO  → Can it tolerate interruption (2-minute warning)?
          ├── YES → Spot Instances (up to 90% off!)
          │         Use for: batch jobs, CI/CD, data processing
          └── NO  → On-Demand (full price, no commitment)
                    Use for: unpredictable, short-term, new workloads
```

## Common Cost Traps

| Trap | Monthly Cost When Forgotten | Prevention |
|------|---------------------------|------------|
| Unused Elastic IP | $3.65/month each | Release unattached EIPs |
| NAT Gateway (idle) | $32.40/month + data charges | Delete in dev environments |
| Stopped RDS (auto-restarts after 7 days) | Full hourly rate | Delete dev databases or use snapshots |
| EBS volumes without instance | $0.08-0.125/GB/month | Enable Delete on Termination |
| Unused ALB | $16.20/month minimum | Delete unused load balancers |
| Old EBS snapshots | $0.05/GB/month | Set lifecycle policies |
| CloudWatch Logs (never expire) | Accumulates forever | Set retention period (14-90 days) |

---

## Quick Reference: Console Navigation

| Task | Path |
|------|------|
| View current bill | Billing → Bills |
| Analyze costs | Billing → Cost Explorer |
| Create budget | Billing → Budgets → Create |
| Enable cost tags | Billing → Cost Allocation Tags |
| Anomaly detection | Billing → Cost Anomaly Detection |
| Buy Savings Plan | Billing → Savings Plans → Purchase |
| View RIs | Billing → Reservations |
| Free tier usage | Billing → Free Tier |
| Price calculator | calculator.aws |
| Compute optimizer | Compute Optimizer (service) |
| Trusted Advisor | Trusted Advisor (service) |

---

## What's Next?

In the next chapter, we'll cover VPC (Virtual Private Cloud) — networking fundamentals, subnets, route tables, security groups, NACLs, NAT, VPN, and VPC peering.

→ Next: [Chapter 6: VPC - Virtual Private Cloud](06-vpc.md)

---

*Last Updated: May 2026*
