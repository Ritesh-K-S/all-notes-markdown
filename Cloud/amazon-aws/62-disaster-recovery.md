# Chapter 62: Disaster Recovery

---

## Table of Contents

- [Overview](#overview)
- [Part 1: RTO and RPO](#part-1-rto-and-rpo)
- [Part 2: Four DR Strategies](#part-2-four-dr-strategies)
- [Part 3: AWS Backup — Portal Walkthrough](#part-3-aws-backup--portal-walkthrough)
- [Part 4: AWS Elastic Disaster Recovery (DRS)](#part-4-aws-elastic-disaster-recovery-drs)
- [Part 5: DR for Key AWS Services](#part-5-dr-for-key-aws-services)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)

---

## Overview

### What is Disaster Recovery? Why Does It Matter?

**Disaster Recovery (DR)** is like a fire drill for your IT systems. A "disaster" could be:
- 🔥 An entire AWS region goes down (rare but possible)
- 💥 Someone accidentally deletes the production database
- 🐛 A software bug corrupts critical data
- 🌊 A natural disaster affects the data center

**Without DR:** Your application is down until you manually rebuild everything (hours, days, or forever if data is lost).
**With DR:** You switch to a backup environment and are back online in minutes.

**The two key numbers you need to decide:**
- **RPO (Recovery Point Objective)**: How much data can you afford to **lose**? (e.g., "we can't lose more than 1 hour of data")
- **RTO (Recovery Time Objective)**: How long can you afford to be **down**? (e.g., "we need to be back online within 30 minutes")

The lower your RPO/RTO requirements, the more expensive your DR solution. A trading platform needs near-zero RPO/RTO ($$$), while a dev environment might not need DR at all.

Disaster Recovery (DR) is the process of preparing for and recovering from events that prevent access to IT resources. AWS provides tools and strategies to implement DR at various cost and recovery-time levels.

```
What you'll learn:
├── RTO (Recovery Time Objective) and RPO (Recovery Point Objective)
├── Four DR strategies: Backup/Restore, Pilot Light, Warm Standby, Multi-Site
├── AWS Backup (centralized backup management)
├── Elastic Disaster Recovery (automated DR)
├── DR for databases, compute, storage
└── Testing and automation best practices
```

---

## Part 1: RTO and RPO

```
┌─────────────────────────────────────────────────────────────────────┐
│           RTO and RPO                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ RPO (Recovery Point Objective):                                     │
│ → How much data can you afford to LOSE?                           │
│ → Measured in time: RPO = 1 hour means max 1 hour of data loss │
│                                                                       │
│ RTO (Recovery Time Objective):                                      │
│ → How long can you afford to be DOWN?                             │
│ → Measured in time: RTO = 4 hours means must recover in 4 hours│
│                                                                       │
│     ←───── RPO ─────→←────────── RTO ──────────→               │
│     Last backup      Disaster    Recovery complete                │
│     ─────────●──────────●──────────────●──────────→ time        │
│              Data loss    Downtime                                  │
│              window       window                                    │
│                                                                       │
│ Cost vs Recovery:                                                    │
│ ├── Lower RTO/RPO = Higher cost                                  │
│ ├── Business determines acceptable RTO/RPO per workload        │
│ └── Different workloads can have different requirements         │
│                                                                       │
│ Examples:                                                            │
│ ├── Email system: RPO = 4h, RTO = 24h (Backup & Restore)     │
│ ├── E-commerce: RPO = 1h, RTO = 1h (Warm Standby)           │
│ ├── Trading platform: RPO ≈ 0, RTO = minutes (Multi-Site)   │
│ └── Dev/test: No DR needed                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Four DR Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│           FOUR DR STRATEGIES (lowest → highest cost)                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. BACKUP & RESTORE                                                 │
│ ┌─────────────────────────────────────────────────┐               │
│ │ Primary Region         │ DR Region              │               │
│ │ [Active workload]      │ [S3 backups only]     │               │
│ │ EC2, RDS, EBS          │ AMIs, snapshots,      │               │
│ │                        │ DB backups in S3      │               │
│ └─────────────────────────────────────────────────┘               │
│ RPO: Hours │ RTO: Hours │ Cost: $ (lowest)                      │
│ → Restore from backups when disaster occurs                     │
│ → Longest recovery time, cheapest                               │
│                                                                       │
│ 2. PILOT LIGHT                                                      │
│ ┌─────────────────────────────────────────────────┐               │
│ │ Primary Region         │ DR Region              │               │
│ │ [Active workload]      │ [Core DB running]     │               │
│ │ EC2, RDS, full stack   │ RDS replica (running) │               │
│ │                        │ AMIs ready            │               │
│ │                        │ Infra as Code ready   │               │
│ └─────────────────────────────────────────────────┘               │
│ RPO: Minutes │ RTO: 10s of minutes │ Cost: $$                 │
│ → Core services always running (DB replication)               │
│ → Scale up remaining infra on failover                         │
│                                                                       │
│ 3. WARM STANDBY                                                     │
│ ┌─────────────────────────────────────────────────┐               │
│ │ Primary Region         │ DR Region              │               │
│ │ [Full scale]           │ [Minimum scale]       │               │
│ │ 3x EC2, RDS Multi-AZ  │ 1x EC2, RDS replica  │               │
│ │ Full ALB               │ ALB running           │               │
│ │                        │ Ready to scale up     │               │
│ └─────────────────────────────────────────────────┘               │
│ RPO: Seconds │ RTO: Minutes │ Cost: $$$                       │
│ → Scaled-down but fully functional copy                        │
│ → Scale up to full capacity on failover                       │
│                                                                       │
│ 4. MULTI-SITE ACTIVE-ACTIVE                                       │
│ ┌─────────────────────────────────────────────────┐               │
│ │ Region A               │ Region B              │               │
│ │ [Full scale - Active]  │ [Full scale - Active] │               │
│ │ Full EC2/ECS fleet     │ Full EC2/ECS fleet    │               │
│ │ Aurora Global (primary)│ Aurora Global (reader)│               │
│ │ DynamoDB Global Table  │ DynamoDB Global Table │               │
│ └─────────────────────────────────────────────────┘               │
│ RPO: Near zero │ RTO: Near zero │ Cost: $$$$ (highest)       │
│ → Both regions serve traffic simultaneously                    │
│ → Route 53 failover routing for automatic switching           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: AWS Backup — Portal Walkthrough

```
Console → AWS Backup

┌─────────────────────────────────────────────────────────────────┐
│ Create Backup Plan                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Start options:                                                  │
│ ○ Start with a template                                        │
│   ├── Daily-35day-retention                                   │
│   ├── Daily-Monthly-1yr-retention                            │
│   └── Custom                                                   │
│ ● Build a new plan                                             │
│ ○ Define plan using JSON                                      │
│                                                                   │
│ Backup plan name: [production-backup-plan]                    │
│                                                                   │
│ Backup rule 1:                                                  │
│ ├── Rule name: [daily-backup]                                │
│ ├── Backup vault: [Default ▼] or create new                 │
│ │   → Vault = encrypted container for backups               │
│ │   → KMS key: [aws/backup ▼] or custom key                │
│ ├── Backup frequency:                                         │
│ │   ● Daily                                                   │
│ │   ○ Weekly                                                  │
│ │   ○ Monthly                                                 │
│ │   ○ Custom cron expression                                 │
│ ├── Backup window:                                            │
│ │   ● Use default (5 AM UTC, 8-hour window)                │
│ │   ○ Customize: Start [02:00 UTC] Window [4 hours]        │
│ ├── Transition to cold storage: [30] days (optional)       │
│ ├── Retention period: [35] days                              │
│ ├── Copy to another region: ☑                                │
│ │   ├── Destination region: [eu-west-1 ▼]                  │
│ │   └── Retention in destination: [35] days                │
│ └── Enable continuous backup: ☑ (point-in-time recovery)  │
│     → Supported: RDS, S3, DynamoDB, EFS                     │
│                                                                   │
│ Backup rule 2 (optional):                                     │
│ ├── Rule name: [monthly-backup]                              │
│ ├── Frequency: Monthly (1st of month)                       │
│ └── Retention: [365] days                                    │
│                                                                   │
│                         [Create plan]                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Assign Resources                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Resource assignment name: [production-resources]              │
│                                                                   │
│ IAM role: ● Default role ○ Choose IAM role                  │
│                                                                   │
│ Resource selection:                                            │
│ ● Include all resource types                                  │
│ ○ Include specific resource types:                            │
│   ☑ EC2  ☑ EBS  ☑ RDS  ☑ DynamoDB                         │
│   ☑ EFS  ☑ S3   ☐ FSx  ☐ Aurora                           │
│                                                                   │
│ Refine by tags:                                                │
│ Key: [Environment]  Value: [Production]                      │
│ → Only backup resources with this tag                        │
│                                                                   │
│                         [Assign resources]                    │
└─────────────────────────────────────────────────────────────────┘

Supported services:
├── EC2 (AMIs), EBS (snapshots), RDS (snapshots), Aurora
├── DynamoDB, EFS, FSx, S3, Storage Gateway
├── DocumentDB, Neptune, Timestream, CloudFormation
├── VMware VMs (on-premises via Backup Gateway)
└── SAP HANA on EC2

⚡ Cross-account backup: Share vault with other accounts (Organizations)
⚡ Backup Audit Manager: Compliance reporting
⚡ Vault Lock: WORM (Write Once Read Many) for compliance
```

---

## Part 4: AWS Elastic Disaster Recovery (DRS)

```
Console → AWS Elastic Disaster Recovery

┌─────────────────────────────────────────────────────────────────────┐
│           ELASTIC DISASTER RECOVERY (DRS)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Automated, cost-effective DR for on-prem + cloud servers   │
│ Replaces: CloudEndure Disaster Recovery                            │
│                                                                       │
│ How it works:                                                        │
│ 1. Install AWS Replication Agent on source servers                │
│ 2. Continuous block-level replication to staging area            │
│ 3. Lightweight staging (low-cost instances + EBS)               │
│ 4. Drill/Failover: Launch full-sized recovery instances        │
│ 5. Failback: Reverse replication when primary is recovered     │
│                                                                       │
│ ┌──────────────┐     continuous      ┌─────────────────┐         │
│ │ Source Server  │ ──── replication ─→ │ Staging Area     │         │
│ │ (on-prem/cloud)│    (block-level)   │ (lightweight EC2 │         │
│ └──────────────┘                      │  + EBS volumes)  │         │
│                                        └────────┬────────┘         │
│                                                 │ drill/failover  │
│                                        ┌────────▼────────┐         │
│                                        │ Recovery Instance │         │
│                                        │ (right-sized EC2) │         │
│                                        └─────────────────┘         │
│                                                                       │
│ Setup:                                                               │
│ Console → DRS → Set up replication                                │
│ ├── Replication settings:                                         │
│ │   ├── Staging area subnet: [dr-vpc-subnet ▼]                │
│ │   ├── Replication server type: [t3.small ▼]                 │
│ │   ├── EBS volume type: [gp3 ▼]                              │
│ │   ├── Data routing: Public IP / Private IP (VPN/DX)       │
│ │   └── Bandwidth throttling: [0 (unlimited) ▼]              │
│ ├── Launch settings (per server):                               │
│ │   ├── Instance type: [same as source / custom ▼]           │
│ │   ├── Subnet: [recovery-subnet ▼]                           │
│ │   ├── Security groups: [dr-sg ▼]                            │
│ │   └── IAM role: [optional ▼]                                │
│ ├── Recovery:                                                     │
│ │   ├── Initiate drill: Test failover (non-disruptive)       │
│ │   ├── Initiate failover: Actual recovery                   │
│ │   └── Failback: Reverse replication to original site       │
│ └── Point-in-time recovery: Select recovery point             │
│     → Snapshots taken every few seconds                        │
│                                                                       │
│ Pricing:                                                             │
│ ├── Per source server: $0.028/hr (~$20/month)                 │
│ ├── Staging area: Low-cost t3.small + EBS (minimal)          │
│ ├── Recovery instances: EC2 pricing only during drill/failover│
│ └── No charge for data transfer (replication)                 │
│                                                                       │
│ ⚡ RPO: Seconds (continuous replication)                          │
│ ⚡ RTO: Minutes (launch recovery instances)                      │
│ ⚡ Supports: Linux, Windows (most x86 OS versions)              │
│ ⚡ Regular drills without impacting production                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: DR for Key AWS Services

```
┌─────────────────────────────────────────────────────────────────────┐
│           DR FOR KEY AWS SERVICES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Service           │ DR Mechanism                                    │
│ ──────────────────┼─────────────────────────────────────────────────│
│ EC2               │ AMIs copied cross-region, DRS for continuous  │
│ EBS               │ Snapshots → copy to DR region                 │
│ S3                │ Cross-Region Replication (CRR), versioning    │
│ RDS               │ Multi-AZ (HA), cross-region read replicas    │
│                   │ Automated snapshots → copy cross-region       │
│ Aurora            │ Aurora Global Database (< 1s replication)    │
│                   │ Managed cross-region failover                  │
│ DynamoDB          │ Global Tables (multi-region active-active)   │
│                   │ Point-in-time recovery (PITR) + on-demand backup│
│ EFS               │ Cross-region replication (EFS Replication)   │
│ Lambda            │ Deploy to multiple regions (IaC)              │
│ API Gateway       │ Regional endpoint in each region              │
│ Route 53          │ Failover routing + health checks              │
│ Secrets Manager   │ Replica secrets in DR region                  │
│ CloudFormation    │ StackSets for multi-region deployment        │
│                                                                       │
│ ⚡ Route 53 is the key to automated failover                     │
│ ⚡ Test failover regularly — untested DR is not DR              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
# AWS Backup Plan
resource "aws_backup_plan" "production" {
  name = "production-backup-plan"

  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.production.name
    schedule          = "cron(0 5 * * ? *)"

    lifecycle {
      cold_storage_after = 30
      delete_after       = 365
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr_region.arn
      lifecycle {
        delete_after = 365
      }
    }
  }
}

# Backup selection by tag
resource "aws_backup_selection" "production" {
  iam_role_arn = aws_iam_role.backup.arn
  name         = "production-resources"
  plan_id      = aws_backup_plan.production.id

  selection_tag {
    type  = "STRINGEQUALS"
    key   = "Environment"
    value = "Production"
  }
}

# Backup Vault with KMS encryption
resource "aws_backup_vault" "production" {
  name        = "production-vault"
  kms_key_arn = aws_kms_key.backup.arn
}

# S3 Cross-Region Replication
resource "aws_s3_bucket_replication_configuration" "dr" {
  bucket = aws_s3_bucket.primary.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

```bash
# Create on-demand backup
aws backup start-backup-job \
  --backup-vault-name production-vault \
  --resource-arn arn:aws:rds:us-east-1:123456789012:db:mydb \
  --iam-role-arn arn:aws:iam::123456789012:role/BackupRole

# List recovery points
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name production-vault

# Copy EBS snapshot to DR region
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0123456789abcdef0 \
  --destination-region us-west-2

# Copy AMI to DR region
aws ec2 copy-image \
  --source-region us-east-1 \
  --source-image-id ami-0123456789abcdef0 \
  --name "DR-copy-webserver" \
  --region us-west-2

# Route 53 health check
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "IPAddress": "203.0.113.1",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FailureThreshold": 3
  }'
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD DR PATTERNS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Small Business (Backup & Restore)                       │
│ AWS Backup → Daily backups → Copy to DR region                   │
│ → Manual recovery from backups if disaster occurs                │
│ → RTO: Hours | RPO: 24 hours | Cost: Minimal                    │
│                                                                       │
│ Pattern 2: Mid-Size (Pilot Light)                                   │
│ Primary: us-east-1 (full production)                               │
│ DR: us-west-2 (Aurora cross-region replica + AMIs)               │
│ → Failover: Promote Aurora replica, launch EC2 from AMIs       │
│ → Route 53 failover routing (automatic DNS switch)             │
│ → RTO: 30 min | RPO: Minutes | Cost: Moderate                  │
│                                                                       │
│ Pattern 3: Enterprise (Warm Standby)                                │
│ Primary: us-east-1 (full scale)                                    │
│ DR: eu-west-1 (25% scale, running)                               │
│ → Auto Scaling scales up DR on failover                          │
│ → Aurora Global Database (< 1s replication)                     │
│ → Route 53 weighted/failover routing                            │
│ → RTO: Minutes | RPO: Seconds | Cost: High                     │
│                                                                       │
│ Pattern 4: Critical Systems (Multi-Site Active-Active)            │
│ Both regions serve traffic simultaneously                         │
│ → DynamoDB Global Tables (active-active)                        │
│ → Aurora Global Database with write forwarding                 │
│ → Route 53 latency-based routing                                │
│ → RTO: Near zero | RPO: Near zero | Cost: 2x                  │
│                                                                       │
│ DR Testing Best Practices:                                          │
│ ├── Test DR at least quarterly                                   │
│ ├── Use DRS drill (non-disruptive test failover)              │
│ ├── Automate failover with runbooks (Systems Manager)         │
│ ├── Document and time each recovery step                       │
│ ├── GameDay exercises with the whole team                      │
│ └── Verify data integrity after every drill                    │
│                                                                       │
│ ⚠️ An untested DR plan is not a DR plan                           │
│ ⚡ Start with Backup & Restore, evolve as business requires    │
│ ⚡ Different workloads can use different DR strategies          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Disaster Recovery Quick Reference:
├── RTO: How long can you be down? | RPO: How much data can you lose?
├── Four strategies (lowest → highest cost):
│   ├── 1. Backup & Restore: RPO hours, RTO hours ($)
│   ├── 2. Pilot Light: RPO minutes, RTO 10s of min ($$)
│   ├── 3. Warm Standby: RPO seconds, RTO minutes ($$$)
│   └── 4. Multi-Site Active-Active: RPO ~0, RTO ~0 ($$$$)
├── AWS Backup:
│   ├── Centralized backup for 15+ AWS services
│   ├── Cross-region + cross-account copy
│   ├── Vault Lock (WORM) for compliance
│   └── Continuous backup for PITR (RDS, S3, DynamoDB, EFS)
├── Elastic Disaster Recovery (DRS):
│   ├── Continuous replication (RPO seconds)
│   ├── Non-disruptive drills
│   ├── Point-in-time recovery
│   └── ~$20/month per server + staging costs
├── Key services for DR:
│   ├── Route 53: Failover routing + health checks
│   ├── Aurora Global Database: < 1s cross-region replication
│   ├── DynamoDB Global Tables: Multi-region active-active
│   └── S3 CRR: Cross-region replication
├── ⚡ Test DR quarterly — untested DR is not DR
├── ⚡ Start with Backup & Restore, evolve as needed
└── ⚡ Route 53 = key to automated failover
```

---

**🎉 Congratulations!** You've completed the AWS learning path covering 62 chapters from fundamentals to advanced architecture patterns. Continue practicing in the AWS console, build real projects, and consider pursuing AWS certifications (Cloud Practitioner → Solutions Architect Associate → Professional).
