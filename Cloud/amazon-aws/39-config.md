# Chapter 39: AWS Config

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AWS Config Fundamentals](#part-1-aws-config-fundamentals)
- [Part 2: Setting Up AWS Config (Full Portal Walkthrough)](#part-2-setting-up-aws-config-full-portal-walkthrough)
- [Part 3: Config Rules](#part-3-config-rules)
- [Part 4: Conformance Packs](#part-4-conformance-packs)
- [Part 5: Remediation](#part-5-remediation)
- [Part 6: Advanced Queries & Aggregator](#part-6-advanced-queries--aggregator)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is AWS Config? Why Do We Need Configuration Compliance?

Imagine you have a rule: "All front doors must be locked at night." A **home inspector** regularly checks every door and tells you if any are unlocked. That's what AWS Config does for your cloud infrastructure.

**AWS Config is a compliance checker** that continuously monitors your AWS resources and tells you:
- 📝 **What exists**: A complete inventory of all your AWS resources
- 🔄 **What changed**: "Someone modified the security group at 2 PM yesterday"
- ✅ **Is it compliant?**: "Rule says all S3 buckets must be encrypted — 3 buckets are NOT encrypted"

**CloudWatch vs CloudTrail vs Config — What's the difference?**
- **CloudWatch**: How is my resource **performing**? (CPU, memory, errors)
- **CloudTrail**: **Who** made changes? (audit trail of API calls)
- **Config**: **What** is my resource's **current state** and is it **compliant**? (configuration history)

AWS Config continuously monitors and records your AWS resource configurations and evaluates them against desired configurations. It answers: "What does my infrastructure look like? Has it changed? Is it compliant?"

```
What you'll learn:
├── AWS Config Fundamentals (recording, timeline)
├── Setting up Config (full portal walkthrough)
├── Config Rules (managed, custom, evaluation)
├── Conformance Packs (compliance frameworks)
├── Auto-remediation (fix non-compliant resources)
├── Advanced queries & multi-account aggregation
└── Terraform & CLI examples
```

---

## Part 1: AWS Config Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW AWS CONFIG WORKS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────────┐                                                 │
│ │  AWS Resources  │                                                 │
│ │ EC2, S3, RDS,   │──► Config Recorder ──┬──► S3 (history)       │
│ │ IAM, VPC, etc.  │    (detects changes) ├──► SNS (notifications)│
│ └─────────────────┘                      └──► Config Rules       │
│                                               (evaluate compliance)│
│                                                                       │
│ What Config provides:                                               │
│ ├── Configuration recording: Current state of all resources     │
│ ├── Configuration history: How resource changed over time       │
│ ├── Configuration timeline: Visual history of changes           │
│ ├── Relationships: How resources relate to each other           │
│ ├── Compliance evaluation: Rules to check configurations       │
│ └── Remediation: Auto-fix non-compliant resources               │
│                                                                       │
│ Config vs CloudTrail:                                               │
│ ┌──────────────────┬────────────────────┬───────────────────────┐ │
│ │                  │ AWS Config         │ CloudTrail            │ │
│ ├──────────────────┼────────────────────┼───────────────────────┤ │
│ │ Tracks           │ Resource config    │ API calls             │ │
│ │ Answers          │ "What does it look │ "Who changed it?"     │ │
│ │                  │  like now?"        │                       │ │
│ │ Focus            │ Configuration state│ Activity/audit        │ │
│ │ Compliance       │ Yes (rules)        │ No                    │ │
│ │ Timeline         │ Config changes     │ API call history      │ │
│ └──────────────────┴────────────────────┴───────────────────────┘ │
│                                                                       │
│ Pricing:                                                             │
│ ├── Configuration items recorded: $0.003/item                   │
│ ├── Config rule evaluations: $0.001/evaluation                  │
│ ├── Conformance pack evaluations: $0.001/evaluation             │
│ └── Advanced queries: $0.0010/query                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Setting Up AWS Config (Full Portal Walkthrough)

```
Console → AWS Config → Get started / Settings

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: RECORDING SETTINGS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Recording strategy:                                             │
│ ● Record all current and future resource types                │
│   ☑ Include globally recorded resource types (IAM, etc.)     │
│ ○ Record specific resource types                              │
│   ☑ AWS::EC2::Instance                                        │
│   ☑ AWS::S3::Bucket                                           │
│   ☑ AWS::IAM::Role                                            │
│   → Select specific types to reduce costs                    │
│                                                                   │
│ Recording frequency:                                           │
│ ● Continuous recording (real-time changes)                    │
│ ○ Daily recording (once per 24 hours)                         │
│ → ⚡ Continuous for production, Daily for cost savings        │
│                                                                   │
│ Service-linked role:                                           │
│ ● Use existing service-linked role                            │
│ ○ Choose a role from your account                             │
│ → AWSServiceRoleForConfig (auto-created)                     │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: DELIVERY CHANNEL                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Amazon S3 bucket:                                              │
│ ● Create a bucket                                              │
│ ○ Choose a bucket from your account                           │
│ ○ Choose a bucket from another account                        │
│ Bucket name: [config-bucket-123456789012]                     │
│ Prefix: [AWSConfig/]                                          │
│                                                                   │
│ Amazon SNS topic: ☑ Enable                                    │
│ ● Create a topic                                              │
│ Topic name: [config-notifications]                            │
│ → Notifications on configuration changes and compliance      │
│                                                                   │
│ Delivery frequency: [24 hours ▼]                              │
│ → How often config snapshots are delivered to S3             │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: RULES (optional, add later)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Add rules from AWS managed rules:                              │
│ Search: [encrypted]                                            │
│ ☑ s3-bucket-server-side-encryption-enabled                    │
│ ☑ rds-storage-encrypted                                       │
│ ☑ encrypted-volumes                                           │
│ ☑ cloud-trail-encryption-enabled                              │
│                                                                   │
│                    [Next → Confirm]                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Config Rules

```
Console → AWS Config → Rules → Add rule

┌─────────────────────────────────────────────────────────────────────┐
│           CONFIG RULES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Rule type:                                                           │
│ ● AWS managed rule (200+ pre-built rules)                          │
│ ○ Custom rule (Lambda function)                                    │
│ ○ Custom rule (Guard — policy-as-code)                            │
│                                                                       │
│ Popular managed rules:                                               │
│                                                                       │
│ Security:                                                            │
│ ├── s3-bucket-public-read-prohibited                             │
│ ├── s3-bucket-server-side-encryption-enabled                    │
│ ├── rds-instance-public-access-check                             │
│ ├── restricted-ssh (no 0.0.0.0/0 on port 22)                   │
│ ├── iam-password-policy                                          │
│ ├── iam-root-access-key-check                                    │
│ ├── root-account-mfa-enabled                                     │
│ ├── mfa-enabled-for-iam-console-access                          │
│ └── cloud-trail-enabled                                          │
│                                                                       │
│ Encryption:                                                          │
│ ├── encrypted-volumes (EBS)                                      │
│ ├── rds-storage-encrypted                                        │
│ ├── s3-default-encryption-kms                                    │
│ └── cloud-trail-encryption-enabled                               │
│                                                                       │
│ Networking:                                                          │
│ ├── vpc-flow-logs-enabled                                        │
│ ├── vpc-default-security-group-closed                           │
│ └── restricted-common-ports                                      │
│                                                                       │
│ Rule details:                                                        │
│ Name: [s3-bucket-public-read-prohibited]                           │
│ Trigger type:                                                        │
│ ● Configuration changes (evaluated when resource changes)        │
│ ○ Periodic (evaluated every 1h, 3h, 6h, 12h, 24h)             │
│ Scope: Resource type [AWS::S3::Bucket]                             │
│ Parameters: (rule-specific, e.g., maxAccessKeyAge = 90)           │
│                                                                       │
│ Evaluation results:                                                  │
│ ├── COMPLIANT: Resource meets the rule                           │
│ ├── NON_COMPLIANT: Resource violates the rule                   │
│ └── NOT_APPLICABLE: Rule doesn't apply to this resource         │
│                                                                       │
│ Custom rule with Lambda:                                            │
│ ├── Write Lambda function that evaluates resource config        │
│ ├── Returns COMPLIANT or NON_COMPLIANT                          │
│ ├── Can evaluate any custom business logic                      │
│ └── Example: "All EC2 instances must have tag 'CostCenter'"    │
│                                                                       │
│ Custom rule with Guard:                                             │
│ ├── Policy-as-code using Guard DSL                               │
│ ├── No Lambda needed                                              │
│ └── Example:                                                      │
│     rule s3_encrypted {                                            │
│       resourceType == "AWS::S3::Bucket"                           │
│       configuration.ServerSideEncryptionConfiguration EXISTS    │
│     }                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Conformance Packs

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONFORMANCE PACKS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → AWS Config → Conformance packs → Deploy                  │
│                                                                       │
│ What: Collection of Config rules + remediation actions bundled   │
│ into a compliance framework.                                        │
│                                                                       │
│ Template source:                                                     │
│ ● Use sample template                                               │
│ ○ Use template from S3                                              │
│ ○ Enter template in editor                                          │
│                                                                       │
│ Pre-built packs:                                                     │
│ ├── CIS AWS Foundations Benchmark                                │
│ ├── AWS Best Practices for Security                             │
│ ├── PCI DSS                                                       │
│ ├── HIPAA                                                         │
│ ├── NIST 800-53                                                   │
│ ├── SOC 2                                                         │
│ ├── AWS Operational Best Practices                              │
│ └── Custom packs (YAML template)                                 │
│                                                                       │
│ Example: CIS Benchmark pack evaluates:                             │
│ ├── Root account MFA enabled                                     │
│ ├── IAM password policy                                           │
│ ├── CloudTrail enabled in all regions                            │
│ ├── S3 buckets not public                                         │
│ ├── VPC flow logs enabled                                         │
│ ├── Default security group restricts all traffic               │
│ └── ...40+ more rules                                             │
│                                                                       │
│ Compliance score: Dashboard shows % compliant across pack        │
│                                                                       │
│ ⚡ Use conformance packs for:                                       │
│ ├── Regulatory compliance (PCI, HIPAA, SOC 2)                   │
│ ├── Organization-wide standards                                  │
│ └── Deploy via StackSets to all accounts                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Remediation

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTO-REMEDIATION                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Config → Rules → [Rule] → Actions → Manage remediation │
│                                                                       │
│ Remediation action:                                                  │
│ ● AWS Systems Manager Automation (pre-built documents)            │
│ ○ Custom SSM Automation document                                   │
│                                                                       │
│ Pre-built remediation examples:                                     │
│ ├── AWS-EnableS3BucketEncryption                                │
│ ├── AWS-DisablePublicAccessForSecurityGroup                     │
│ ├── AWS-EnableCloudTrail                                         │
│ ├── AWS-EnableEbsEncryptionByDefault                            │
│ ├── AWS-EnableRDSInstanceDeletionProtection                     │
│ └── AWS-ConfigureS3BucketPublicAccessBlock                      │
│                                                                       │
│ Execution:                                                           │
│ ● Manual remediation (review before fixing)                       │
│ ○ Automatic remediation (⚠️ auto-fix without review)             │
│                                                                       │
│ Auto-remediation:                                                    │
│ Retry attempts: [5]                                                 │
│ Retry interval: [60] seconds                                       │
│                                                                       │
│ → Non-compliant S3 bucket detected                                │
│ → Config triggers SSM Automation                                  │
│ → SSM enables encryption on the bucket                           │
│ → Config re-evaluates → COMPLIANT                                │
│                                                                       │
│ ⚡ Start with manual remediation, move to auto after testing     │
│ ⚠️ Auto-remediation can break things if misconfigured            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Advanced Queries & Aggregator

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADVANCED QUERIES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Config → Advanced queries                                 │
│                                                                       │
│ SQL-like queries across ALL recorded resources:                    │
│                                                                       │
│ # Find all public S3 buckets                                       │
│ SELECT resourceId, resourceName, configuration                    │
│ WHERE resourceType = 'AWS::S3::Bucket'                             │
│ AND configuration.publicAccessBlockConfiguration IS NULL          │
│                                                                       │
│ # Find unencrypted EBS volumes                                     │
│ SELECT resourceId, availabilityZone, configuration.encrypted     │
│ WHERE resourceType = 'AWS::EC2::Volume'                            │
│ AND configuration.encrypted = false                                │
│                                                                       │
│ # Count resources by type                                          │
│ SELECT resourceType, COUNT(*) as resourceCount                    │
│ WHERE resourceType LIKE 'AWS::EC2::%'                              │
│ GROUP BY resourceType                                               │
│ ORDER BY resourceCount DESC                                        │
│                                                                       │
│ # Find EC2 instances without specific tag                          │
│ SELECT resourceId, resourceName, tags                              │
│ WHERE resourceType = 'AWS::EC2::Instance'                          │
│ AND tags.tag('CostCenter') IS NULL                                │
│                                                                       │
│ ── AGGREGATOR ──                                                    │
│ Console → Config → Aggregators → Add aggregator                   │
│                                                                       │
│ Aggregates Config data across:                                     │
│ ├── Multiple accounts (specify account IDs or Organization)    │
│ ├── Multiple regions                                              │
│ └── Single dashboard for all accounts+regions                    │
│                                                                       │
│ Source type:                                                         │
│ ● Add individual account IDs                                       │
│ ○ Add my organization (all accounts)                              │
│ Regions: ☑ All regions                                             │
│                                                                       │
│ ⚡ Required for multi-account compliance visibility              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & CLI Examples

```hcl
resource "aws_config_configuration_recorder" "main" {
  name     = "default"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "default"
  s3_bucket_name = aws_s3_bucket.config.id
  sns_topic_arn  = aws_sns_topic.config.arn

  snapshot_delivery_properties {
    delivery_frequency = "TwentyFour_Hours"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_config_rule" "s3_encrypted" {
  name = "s3-bucket-server-side-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_remediation_configuration" "s3_encrypted" {
  config_rule_name = aws_config_config_rule.s3_encrypted.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-EnableS3BucketEncryption"

  parameter {
    name           = "BucketName"
    resource_value = "RESOURCE_ID"
  }

  automatic                  = true
  maximum_automatic_attempts = 5
  retry_attempt_seconds      = 60
}
```

```bash
# Get resource configuration
aws configservice get-resource-config-history \
  --resource-type AWS::S3::Bucket \
  --resource-id my-bucket

# Get compliance by rule
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-server-side-encryption-enabled \
  --compliance-types NON_COMPLIANT

# Run advanced query
aws configservice select-resource-config \
  --expression "SELECT resourceId WHERE resourceType = 'AWS::EC2::Instance'"

# Evaluate rule manually
aws configservice start-config-rules-evaluation \
  --config-rule-names s3-bucket-server-side-encryption-enabled
```

---

## Real-World Patterns

### Pattern 1: Compliance Dashboard for Security Audit

```
Goal: Show auditors that all resources meet security standards

Setup:
1. Enable Conformance Pack: "AWS Security Best Practices"
   (Bundles 50+ rules: encryption, public access, logging)
2. Set up Config Aggregator across all accounts
3. Use Config Dashboard → Compliance overview
4. Export: Download compliance report as CSV

Auditor sees:
┌─────────────────────────────────────────────────────────┐
│ Compliance Status                                        │
│                                                          │
│ Compliant:     487 resources  (94%)  ████████████████░░  │
│ Non-compliant:  31 resources  (6%)   ███░░░░░░░░░░░░░░  │
│                                                          │
│ Top violations:                                          │
│ ├── 12 × S3 buckets without versioning                   │
│ ├── 8  × Security groups with 0.0.0.0/0 SSH              │
│ ├── 6  × EBS volumes unencrypted                         │
│ └── 5  × RDS without deletion protection                 │
└─────────────────────────────────────────────────────────┘
```

### Pattern 2: Auto-Remediation

```
Scenario: Automatically fix non-compliant resources

Rule: s3-bucket-server-side-encryption-enabled
  → NON_COMPLIANT? → SSM Automation → Enable encryption automatically

Setup:
Config Rule → Remediation action → AWS-EnableS3BucketEncryption
  Automatic remediation: ✅ Enable
  Retry: 3 attempts, 60 seconds between

Result: Unencrypted S3 bucket detected → Auto-encrypted in <5 minutes
```

### Pattern 3: Change Investigation (Who Changed What?)

```
Scenario: "Our production RDS security group was changed — who did it?"

Investigation:
1. Console → Config → Resources → Select the security group
2. View Resource Timeline → See all configuration changes
3. Click the change event → See exactly what changed (before/after)
4. Cross-reference with CloudTrail → Find the user who made the API call

Timeline:
  2025-01-15 10:00 — sg-prod-rds rule added: 0.0.0.0/0 port 3306 ← 🚨
  2025-01-14 09:30 — sg-prod-rds: no changes (compliant)
```

> ⚠️ **Config pricing:** $0.003 per configuration item recorded + $0.001 per rule evaluation. Enable only rules you'll actually act on to control costs.

---

## Quick Reference

```
AWS Config Quick Reference:
├── What: Resource configuration recorder + compliance evaluator
├── Records: Current state + history of all resource configs
├── Config Rules: 200+ managed rules + custom Lambda/Guard rules
├── Evaluation: COMPLIANT / NON_COMPLIANT per resource per rule
├── Conformance Packs: CIS, PCI, HIPAA, SOC 2, NIST bundles
├── Remediation: Auto-fix via SSM Automation documents
├── Timeline: Visual history of config changes per resource
├── Advanced queries: SQL across all resources
├── Aggregator: Multi-account + multi-region view
├── Config vs CloudTrail: Config=WHAT, CloudTrail=WHO
├── Pricing: $0.003/config item, $0.001/rule evaluation
└── ⚡ Start with CIS Benchmark conformance pack
```

---

## What's Next?

In **Chapter 40: KMS**, we'll begin the Security & Compliance section — covering key management, encryption, key policies, and data protection across AWS services.
