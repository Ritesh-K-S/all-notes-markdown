# Chapter 61: AWS Cost Optimization

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AWS Cost Management Tools](#part-1-aws-cost-management-tools)
- [Part 2: Cost Explorer — Portal Walkthrough](#part-2-cost-explorer--portal-walkthrough)
- [Part 3: AWS Budgets — Portal Walkthrough](#part-3-aws-budgets--portal-walkthrough)
- [Part 4: Savings Plans & Reserved Instances](#part-4-savings-plans--reserved-instances)
- [Part 5: Spot Instances & Other Discounts](#part-5-spot-instances--other-discounts)
- [Part 6: Practical Cost Reduction Strategies](#part-6-practical-cost-reduction-strategies)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### How Does AWS Billing Work? Why Does Cost Optimization Matter?

AWS uses a **pay-as-you-go** model — you pay only for what you use, like an electricity bill. But just like leaving lights on in every room runs up your electricity bill, leaving unused AWS resources running wastes money.

**How you get charged:**
- ⏰ **Compute**: Per hour/second your EC2 instances run
- 🗄️ **Storage**: Per GB stored in S3, EBS, etc.
- 🌐 **Data transfer**: Per GB of data leaving AWS (incoming is free)
- 📞 **Requests**: Per API call, query, or request (varies by service)

**Common cost surprises for beginners:**
- Forgot to stop a dev EC2 instance over the weekend (~$2-20 wasted)
- NAT Gateway data charges (can be $30+/month even for small traffic)
- EBS volumes left attached after terminating EC2 instances
- Old snapshots accumulating over months

**Where to see your bill:** Console → Billing Dashboard (top-right menu → Billing)

Cost optimization is an ongoing process. AWS provides tools to understand, control, and reduce spending while maintaining performance and reliability.

```
What you'll learn:
├── Cost Explorer, Budgets, Cost Anomaly Detection
├── Savings Plans vs Reserved Instances
├── Spot Instances for up to 90% savings
├── Right-sizing with Compute Optimizer
├── Practical strategies to reduce your AWS bill
└── Cost allocation and tagging
```

---

## Part 1: AWS Cost Management Tools

```
┌─────────────────────────────────────────────────────────────────────┐
│           COST MANAGEMENT TOOLS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Tool                     │ Purpose                                  │
│ ─────────────────────────┼──────────────────────────────────────────│
│ Cost Explorer            │ Visualize, analyze, forecast costs      │
│ Budgets                  │ Set spending thresholds + alerts         │
│ Cost Anomaly Detection   │ ML-based unusual spending alerts        │
│ Cost & Usage Report (CUR)│ Detailed line-item billing data → S3  │
│ Pricing Calculator       │ Estimate costs before deploying         │
│ Compute Optimizer        │ Right-sizing recommendations            │
│ Trusted Advisor          │ Cost optimization checks (5 free)      │
│ Billing Dashboard        │ Month-to-date summary                   │
│ Cost Allocation Tags     │ Tag resources → track cost by project │
│                                                                       │
│ Free Tier tracking:                                                  │
│ Console → Billing → Free Tier                                     │
│ → Shows usage vs free tier limits per service                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Cost Explorer — Portal Walkthrough

```
Console → Billing → Cost Explorer

┌─────────────────────────────────────────────────────────────────┐
│ AWS Cost Explorer                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Date range: [Last 6 months ▼]                                 │
│ Granularity: ○ Daily ● Monthly ○ Hourly                     │
│                                                                   │
│ Group by:                                                      │
│ ├── Service → see cost per AWS service                       │
│ ├── Region → cost per region                                 │
│ ├── Linked account → cost per account (Organizations)       │
│ ├── Tag → cost per project/team (requires tag activation)  │
│ ├── Instance type → which EC2 types cost most               │
│ ├── Usage type → data transfer, requests, hours             │
│ └── Purchase option → On-Demand vs Reserved vs Spot         │
│                                                                   │
│ Filters:                                                       │
│ ├── Service: [EC2, RDS, S3 ▼]                               │
│ ├── Region: [us-east-1 ▼]                                   │
│ ├── Tag: [Environment = Production ▼]                       │
│ ├── Instance type: [m5.xlarge ▼]                            │
│ └── Purchase option: [On-Demand ▼]                          │
│                                                                   │
│ Views:                                                         │
│ ├── Cost & usage graph (bar/line chart)                      │
│ ├── Savings Plans recommendations                            │
│ │   ├── Lookback period: [60 days ▼]                       │
│ │   ├── Term: ● 1 year ○ 3 year                            │
│ │   ├── Payment: ○ All upfront ● Partial ○ No upfront    │
│ │   └── Shows estimated monthly savings                    │
│ ├── Reserved Instance recommendations                       │
│ ├── Reservation utilization (are you using what you paid?) │
│ └── Forecast (ML-predicted spend for next 3 months)        │
│                                                                   │
│ ⚡ Enable hourly granularity for detailed analysis            │
│ ⚡ Save reports as bookmarks for quick access                │
│ ⚡ Cost Explorer API: Programmatic access to cost data      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: AWS Budgets — Portal Walkthrough

```
Console → Billing → Budgets → Create budget

┌─────────────────────────────────────────────────────────────────┐
│ Create Budget                                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Budget type:                                                    │
│ ● Cost budget     → Track spending ($)                        │
│ ○ Usage budget    → Track usage units (GB, hours)            │
│ ○ Savings Plans budget → Track SP utilization/coverage      │
│ ○ Reservation budget   → Track RI utilization/coverage      │
│                                                                   │
│ Templates (quick setup):                                       │
│ ○ Monthly cost budget                                          │
│ ○ Daily Savings Plans coverage budget                         │
│ ○ Daily reservation utilization budget                       │
│ ● Customize (advanced)                                         │
│                                                                   │
│ Budget details:                                                │
│ Name: [monthly-total-budget]                                  │
│ Period: ● Monthly ○ Quarterly ○ Annually                   │
│ Budget amount: ● Fixed ○ Planned (different per month)     │
│ Amount: [$5000]                                                │
│                                                                   │
│ Scope (filters):                                               │
│ ├── Service: [All services ▼]                                │
│ ├── Linked account: [All accounts ▼]                        │
│ ├── Tag: [All tags ▼] or specific tag key-value             │
│ ├── Region: [All regions ▼]                                  │
│ └── Instance type: [All ▼]                                   │
│                                                                   │
│ Alerts (up to 5):                                              │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ Alert 1:                                                   │ │
│ │ Threshold: [80] % of budgeted amount                     │ │
│ │ Trigger: ● Actual ○ Forecasted                           │ │
│ │                                                           │ │
│ │ Notification:                                              │ │
│ │ ├── Email: [team@company.com, cfo@company.com]          │ │
│ │ ├── SNS topic: [arn:aws:sns:... ▼] (optional)          │ │
│ │ └── Chatbot: [Slack channel ▼] (optional)               │ │
│ └───────────────────────────────────────────────────────────┘ │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ Alert 2:                                                   │ │
│ │ Threshold: [100] % — Forecasted                          │ │
│ └───────────────────────────────────────────────────────────┘ │
│                                                                   │
│ Budget actions (auto-remediation):                             │
│ ├── Action: Apply IAM policy (deny new resources)           │
│ ├── Action: Apply SCP (restrict account)                    │
│ ├── Action: Stop EC2 instances                               │
│ └── Run automatically or require approval                   │
│                                                                   │
│                         [Create budget]                       │
└─────────────────────────────────────────────────────────────────┘

Cost Anomaly Detection:
Console → Billing → Cost Anomaly Detection → Create monitor
├── Monitor type:
│   ● AWS services (all services)
│   ○ Linked account
│   ○ Cost category
│   ○ Cost allocation tag
├── Alert threshold: [$50] minimum impact
├── Frequency: ● Individual alerts ○ Daily/Weekly summary
└── Subscribers: [email / SNS]
→ ML automatically detects unusual spending patterns
```

---

## Part 4: Savings Plans & Reserved Instances

```
┌─────────────────────────────────────────────────────────────────────┐
│           SAVINGS PLANS vs RESERVED INSTANCES                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ SAVINGS PLANS (recommended):                                        │
│                                                                       │
│ Compute Savings Plans:                                              │
│ ├── Commit: $/hr for 1 or 3 years                               │
│ ├── Applies to: EC2, Lambda, Fargate (any region, family, OS) │
│ ├── Discount: Up to 66%                                          │
│ ├── Most flexible — automatically applies to cheapest first   │
│ └── Changes in instance type/region/OS still covered          │
│                                                                       │
│ EC2 Instance Savings Plans:                                        │
│ ├── Commit: $/hr for specific instance family + region         │
│ ├── Discount: Up to 72% (slightly better than Compute SP)    │
│ ├── Locked to: Instance family (e.g., m5) + region            │
│ └── Flexible on: Size, OS, tenancy                             │
│                                                                       │
│ SageMaker Savings Plans:                                            │
│ ├── Same concept, for SageMaker instances                       │
│ └── Up to 64% discount                                          │
│                                                                       │
│ Payment options:                                                     │
│ ├── All Upfront: Highest discount                               │
│ ├── Partial Upfront: Medium discount                            │
│ └── No Upfront: Lowest discount, no commitment cash           │
│                                                                       │
│ RESERVED INSTANCES (legacy, still available):                     │
│ ├── Standard RI: Up to 72%, locked to type + region           │
│ ├── Convertible RI: Up to 66%, can change instance type      │
│ └── Can sell unused RIs on RI Marketplace                      │
│                                                                       │
│ ⚡ Use Savings Plans over RIs for new commitments               │
│ ⚡ Start with Cost Explorer recommendations                     │
│ ⚡ Buy after 1-2 months of stable usage data                   │
│                                                                       │
│ Comparison:                                                          │
│ Feature              │ Compute SP │ EC2 SP   │ Standard RI       │
│ ─────────────────────┼────────────┼──────────┼───────────────────│
│ Max discount         │ 66%        │ 72%      │ 72%               │
│ Applies to Lambda    │ ✅          │ ❌        │ ❌                 │
│ Applies to Fargate   │ ✅          │ ❌        │ ❌                 │
│ Change instance type │ ✅          │ Same fam │ ❌ (Convertible ✅)│
│ Change region        │ ✅          │ ❌        │ ❌                 │
│ Change OS            │ ✅          │ ✅        │ ❌ (Convertible ✅)│
│ Sell unused          │ ❌          │ ❌        │ ✅ (Marketplace)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Spot Instances & Other Discounts

```
┌─────────────────────────────────────────────────────────────────────┐
│           SPOT INSTANCES & OTHER SAVINGS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ EC2 Spot Instances:                                                  │
│ ├── Up to 90% discount vs On-Demand                             │
│ ├── Can be interrupted with 2-minute warning                   │
│ ├── Best for: Batch processing, CI/CD, big data, testing      │
│ ├── Strategies:                                                  │
│ │   ├── Spot Fleet: Mix of Spot + On-Demand                  │
│ │   ├── Diversify across instance types + AZs               │
│ │   ├── Use Spot with Auto Scaling groups                    │
│ │   └── Spot placement score (predict availability)         │
│ └── NOT for: Databases, stateful apps, critical workloads   │
│                                                                       │
│ S3 Storage Savings:                                                  │
│ ├── S3 Intelligent-Tiering: Automatic tier optimization      │
│ ├── Lifecycle policies: Move old data to cheaper tiers       │
│ │   Standard → IA (30d) → Glacier (90d) → Deep (180d)    │
│ └── Delete incomplete multipart uploads                       │
│                                                                       │
│ Data Transfer Savings:                                              │
│ ├── Use CloudFront (cheaper than direct EC2 transfer out)   │
│ ├── VPC endpoints (avoid NAT Gateway data charges)          │
│ ├── Same-AZ traffic is free (use placement awareness)      │
│ └── Use AWS PrivateLink for cross-VPC traffic               │
│                                                                       │
│ Other:                                                               │
│ ├── Graviton instances: 20% cheaper, 40% better performance │
│ ├── Lambda: ARM/Graviton = 20% cheaper                       │
│ ├── GP3 EBS: Better price/perf than GP2                      │
│ ├── NAT Instance (t4g.nano) vs NAT Gateway for dev/test    │
│ └── Scheduled scaling: Scale down nights/weekends           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Practical Cost Reduction Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│           COST REDUCTION CHECKLIST                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Immediate wins (do these first):                                   │
│ ☐ Delete unused EBS volumes and old snapshots                    │
│ ☐ Release unattached Elastic IPs ($3.60/month each)            │
│ ☐ Stop/terminate idle EC2 instances                               │
│ ☐ Delete unused load balancers ($16+/month each)               │
│ ☐ Remove unused NAT Gateways ($32+/month each)                │
│ ☐ Check for idle RDS instances                                    │
│                                                                       │
│ Right-sizing:                                                        │
│ ☐ Review Compute Optimizer recommendations                      │
│ ☐ Downsize over-provisioned instances                            │
│ ☐ Switch to Graviton (arm64) instances                          │
│ ☐ Use GP3 instead of GP2 for EBS                                │
│                                                                       │
│ Commitments:                                                         │
│ ☐ Purchase Savings Plans for stable workloads                   │
│ ☐ Use Spot for fault-tolerant workloads                         │
│ ☐ Reserved capacity for RDS, ElastiCache, Redshift, OpenSearch│
│                                                                       │
│ Architecture changes:                                                │
│ ☐ Use serverless (Lambda) for sporadic workloads               │
│ ☐ S3 lifecycle policies for all buckets                         │
│ ☐ CloudFront for data transfer optimization                    │
│ ☐ VPC endpoints to avoid NAT data charges                      │
│ ☐ Schedule non-prod environments (stop nights/weekends)       │
│                                                                       │
│ Governance:                                                          │
│ ☐ Enable cost allocation tags                                    │
│ ☐ Set budgets with alerts                                        │
│ ☐ Enable Cost Anomaly Detection                                 │
│ ☐ Monthly cost review meetings                                   │
│ ☐ Use SCPs to prevent expensive instance types in dev         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Startup Cost Management                                  │
│ → Use serverless everywhere possible (zero idle cost)            │
│ → Spot for CI/CD and testing                                      │
│ → Set budget alerts at $100, $500, $1000                        │
│ → Enable Cost Anomaly Detection from day one                    │
│                                                                       │
│ Pattern 2: Enterprise FinOps                                        │
│ → Tag strategy: Project, Environment, Team, CostCenter         │
│ → CUR → Athena/QuickSight for detailed cost analytics         │
│ → Compute SP (Compute) for org-wide commitment                 │
│ → Monthly showback reports per business unit                   │
│ → Automated cleanup: Lambda removes tagged dev resources      │
│                                                                       │
│ Pattern 3: Dev/Test Cost Savings                                   │
│ → Schedule: EventBridge → Lambda stops instances at 7 PM      │
│ → Use Spot for all dev/test EC2                                 │
│ → Single-AZ RDS (no Multi-AZ for dev)                          │
│ → Smaller instance types (t3.micro/small)                     │
│ → S3 lifecycle: Delete test data after 30 days                │
│ → Budget per developer account                                 │
│                                                                       │
│ ⚡ Tips:                                                              │
│ 1. Cost optimization is ongoing, not one-time                  │
│ 2. Tag everything from day one                                  │
│ 3. Start with Compute SP after 1-2 months of data            │
│ 4. Review Trusted Advisor cost checks weekly                  │
│ 5. Data transfer is often the hidden cost — optimize early   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Cost Optimization Quick Reference:
├── Tools:
│   ├── Cost Explorer: Analyze spending (group by service/tag/account)
│   ├── Budgets: Alerts at thresholds + auto-actions
│   ├── Cost Anomaly Detection: ML-based unusual spend alerts
│   ├── Compute Optimizer: Right-sizing recommendations
│   └── CUR: Detailed line-item data → S3 → Athena
├── Discounts:
│   ├── Savings Plans: Up to 66-72% (1-3 yr commitment)
│   │   ├── Compute SP: Most flexible (EC2/Lambda/Fargate)
│   │   └── EC2 SP: Better discount, locked to family+region
│   ├── Reserved Instances: Up to 72% (legacy, still works)
│   ├── Spot: Up to 90% (interruptible, fault-tolerant only)
│   └── Graviton: 20% cheaper + 40% better performance
├── Quick wins:
│   ├── Delete unused EBS, EIPs, LBs, NAT GWs
│   ├── Right-size with Compute Optimizer
│   ├── S3 lifecycle policies
│   ├── Schedule dev/test (stop nights/weekends)
│   └── CloudFront for data transfer savings
├── ⚡ Tag everything for cost allocation
├── ⚡ Savings Plans > Reserved Instances (more flexible)
└── ⚡ Cost optimization is continuous, not one-time
```

---

## What's Next?

In **Chapter 62: Disaster Recovery**, we'll cover the four DR strategies (Backup & Restore, Pilot Light, Warm Standby, Multi-Site Active-Active), RTO/RPO planning, AWS Backup, and Elastic Disaster Recovery.
