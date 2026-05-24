# Chapter 37: CloudTrail

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CloudTrail Fundamentals](#part-1-cloudtrail-fundamentals)
- [Part 2: Creating a Trail (Full Portal Walkthrough)](#part-2-creating-a-trail-full-portal-walkthrough)
- [Part 3: Event Types & Event History](#part-3-event-types--event-history)
- [Part 4: CloudTrail Lake & Insights](#part-4-cloudtrail-lake--insights)
- [Part 5: Integration with Other Services](#part-5-integration-with-other-services)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is CloudTrail? Why Do We Need an Audit Trail?

Imagine a building with security cameras at every entrance and elevator. If something goes missing, you can check the footage: **who** entered, **when**, and **what they did**.

**CloudTrail is the security camera for your AWS account.** Every time anyone (or any service) does anything in AWS — launches an EC2 instance, modifies a security group, deletes an S3 bucket — CloudTrail records it as an **event**.

**Why this matters:**
- 🔍 **Security investigation**: "Who deleted the production database at 3 AM?"
- ✅ **Compliance**: Auditors can verify what changes were made and by whom
- 🐛 **Troubleshooting**: "Why did my instance suddenly stop? Let me check CloudTrail"
- 🚨 **Alerting**: Detect suspicious activity (e.g., someone logging in from an unusual country)

AWS CloudTrail records all API calls made in your AWS account. It's the audit log for who did what, when, and from where — essential for security, compliance, and incident investigation.

```
What you'll learn:
├── CloudTrail Fundamentals (events, trails)
├── Creating a trail (full portal walkthrough)
├── Event types (management, data, insights)
├── Event History (free, 90-day lookup)
├── CloudTrail Lake (SQL-based event analysis)
├── CloudTrail Insights (anomaly detection)
├── Integration with CloudWatch, EventBridge, Athena
└── Terraform & CLI examples
```

---

## Part 1: CloudTrail Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW CLOUDTRAIL WORKS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every API call in AWS:                                              │
│ ┌──────────────┐                                                    │
│ │ Console      │                                                    │
│ │ CLI          │──► AWS API ──► CloudTrail ──┬──► S3 (long-term)  │
│ │ SDK          │                              ├──► CloudWatch Logs │
│ │ Terraform    │                              └──► EventBridge     │
│ └──────────────┘                                                    │
│                                                                       │
│ What CloudTrail captures per event:                                 │
│ ├── Who: IAM identity (user, role, service)                      │
│ ├── What: API action (RunInstances, PutObject, etc.)            │
│ ├── When: Timestamp (UTC)                                        │
│ ├── Where: Source IP address, region                             │
│ ├── How: Console, CLI, SDK, service                              │
│ ├── Request parameters: What was requested                       │
│ ├── Response: Success or error + error code                     │
│ └── Resources: ARNs of affected resources                       │
│                                                                       │
│ Free tier:                                                           │
│ ├── Event History: Last 90 days of management events (free!)    │
│ ├── 1 free trail per account for management events              │
│ └── Console lookup, filter by user/event/resource               │
│                                                                       │
│ Pricing:                                                             │
│ ├── Management events (trail): First copy free, $2/100K events │
│ ├── Data events: $0.10/100K events                              │
│ ├── Insights: $0.35/100K events analyzed                        │
│ └── CloudTrail Lake: $2.50/GB ingested, $0.005/GB scanned     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Trail (Full Portal Walkthrough)

```
Console → CloudTrail → Trails → Create trail

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: GENERAL DETAILS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Trail name: [management-trail]                                 │
│                                                                   │
│ Storage location:                                              │
│ ● Create new S3 bucket                                        │
│   Bucket name: [aws-cloudtrail-logs-123456789012-audit]       │
│ ○ Use existing S3 bucket                                      │
│   Bucket name: [existing-trail-bucket ▼]                      │
│   Prefix: [AWSLogs/]                                          │
│                                                                   │
│ Log file SSE-KMS encryption: ☑ Enabled                        │
│ ● New KMS key                                                  │
│   KMS alias: [cloudtrail-key]                                 │
│ ○ Existing KMS key                                            │
│ → ⚡ Always encrypt CloudTrail logs                            │
│                                                                   │
│ Log file validation: ☑ Enabled                                │
│ → Creates digest files to detect log tampering                │
│ → ⚡ Always enable for compliance                              │
│                                                                   │
│ SNS notification delivery: ☐                                  │
│ → Notify when new log files are delivered to S3              │
│                                                                   │
│ CloudWatch Logs:                                               │
│ ☑ Enabled                                                      │
│ Log group: [/aws/cloudtrail/management]                       │
│ IAM role: [CloudTrail_CloudWatchLogs_Role ▼]                  │
│ → Stream events to CloudWatch for real-time alerting         │
│ → ⚡ Critical for security monitoring                          │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: CHOOSE LOG EVENTS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Event type:                                                     │
│ ☑ Management events                                            │
│   → API calls that manage AWS resources                       │
│   → CreateInstance, DeleteBucket, PutRolePolicy              │
│   → ⚡ Always enabled                                          │
│                                                                   │
│   API activity:                                                │
│   ☑ Read (Describe*, List*, Get*)                             │
│   ☑ Write (Create*, Delete*, Update*, Put*)                   │
│   → ⚡ Log both for full audit trail                           │
│                                                                   │
│   Exclude AWS KMS events: ☑                                   │
│   → KMS generates VERY high volume (Decrypt calls)          │
│   → Exclude to reduce costs unless investigating KMS        │
│                                                                   │
│   Exclude Amazon RDS Data API events: ☑                       │
│                                                                   │
│ ☑ Data events                                                  │
│   → Operations ON resources (object-level)                   │
│   → S3: GetObject, PutObject, DeleteObject                   │
│   → Lambda: Invoke                                            │
│   → DynamoDB: GetItem, PutItem                               │
│                                                                   │
│   Data event source: [S3 ▼]                                  │
│   ○ Log all current and future S3 buckets                    │
│   ● Log individual S3 buckets                                 │
│   Bucket: [sensitive-data-bucket ▼]                           │
│   Read: ☑  Write: ☑                                          │
│                                                                   │
│   → ⚠️ Data events are HIGH VOLUME and cost extra             │
│   → Only enable for sensitive/critical resources             │
│                                                                   │
│ ☐ Insights events                                              │
│   → Detects unusual API activity patterns                    │
│   ☑ API call rate  ☑ API error rate                          │
│   → ML-based anomaly detection                               │
│   → Extra cost                                                │
│                                                                   │
│                    [Next → Review → Create trail]               │
└─────────────────────────────────────────────────────────────────┘

Organization trail:
┌─────────────────────────────────────────────────────────────────┐
│ ☑ Enable for all accounts in my organization                   │
│ → Single trail for ALL accounts in AWS Organization          │
│ → Logs delivered to management account S3 bucket             │
│ → ⚡ Recommended for multi-account setups                     │
│ → Member accounts can see trail but can't modify/delete     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Event Types & Event History

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT TYPES                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Management Events (control plane):                                  │
│ ├── Resource management operations                               │
│ ├── Examples: CreateVpc, RunInstances, CreateUser               │
│ ├── Volume: Low-moderate                                         │
│ ├── Cost: First copy free per trail                             │
│ └── ⚡ Always log these                                          │
│                                                                       │
│ Data Events (data plane):                                           │
│ ├── Operations on data within resources                         │
│ ├── S3: GetObject, PutObject, DeleteObject                     │
│ ├── Lambda: Invoke function                                      │
│ ├── DynamoDB: GetItem, PutItem, DeleteItem                     │
│ ├── Volume: Very high (millions/day)                            │
│ ├── Cost: $0.10/100K events                                     │
│ └── Enable selectively for sensitive resources                 │
│                                                                       │
│ Insights Events:                                                    │
│ ├── ML detects unusual patterns in management events           │
│ ├── API call rate anomalies (sudden spike in API calls)        │
│ ├── API error rate anomalies (unusual error rates)             │
│ └── Generates insight event when anomaly detected              │
│                                                                       │
│ Event History (free):                                               │
│ Console → CloudTrail → Event history                              │
│ ├── Last 90 days of management events                            │
│ ├── Filter by: Event name, User name, Resource type, etc.      │
│ ├── Download as CSV or JSON                                      │
│ └── No trail needed! (always on, free)                          │
│                                                                       │
│ Sample event record:                                                │
│ {                                                                    │
│   "eventVersion": "1.08",                                          │
│   "userIdentity": {                                                │
│     "type": "IAMUser",                                              │
│     "arn": "arn:aws:iam::123456:user/admin",                      │
│     "userName": "admin"                                             │
│   },                                                                 │
│   "eventTime": "2024-01-15T10:30:00Z",                            │
│   "eventSource": "ec2.amazonaws.com",                              │
│   "eventName": "TerminateInstances",                               │
│   "sourceIPAddress": "203.0.113.50",                               │
│   "requestParameters": {                                            │
│     "instancesSet": {"items": [{"instanceId": "i-abc123"}]}      │
│   },                                                                 │
│   "responseElements": {"instancesSet": {"items": [...]}}         │
│ }                                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: CloudTrail Lake & Insights

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUDTRAIL LAKE                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed data lake for CloudTrail events with SQL queries.   │
│ Alternative to Athena for querying CloudTrail logs.                │
│                                                                       │
│ Console → CloudTrail → Lake → Create event data store             │
│                                                                       │
│ Name: [security-audit-store]                                       │
│ Retention: [2555] days (7 years, max)                              │
│ Events: ☑ Management ☑ Data events                               │
│ Organization: ☑ All accounts                                      │
│                                                                       │
│ Query (SQL):                                                        │
│ SELECT userIdentity.userName, eventName, eventTime,               │
│        sourceIPAddress                                              │
│ FROM security-audit-store                                          │
│ WHERE eventName = 'ConsoleLogin'                                   │
│   AND eventTime > '2024-01-01'                                    │
│   AND sourceIPAddress NOT LIKE '10.%'                             │
│ ORDER BY eventTime DESC                                            │
│                                                                       │
│ # Find unauthorized API calls                                      │
│ SELECT userIdentity.arn, eventName, errorCode, errorMessage,      │
│        COUNT(*) as failCount                                       │
│ FROM security-audit-store                                          │
│ WHERE errorCode = 'AccessDenied'                                  │
│   AND eventTime > '2024-01-01'                                    │
│ GROUP BY userIdentity.arn, eventName, errorCode, errorMessage     │
│ ORDER BY failCount DESC                                            │
│                                                                       │
│ ⚡ CloudTrail Lake vs Athena:                                       │
│ ├── Lake: Simpler setup, built-in, SQL-based                    │
│ ├── Athena: More flexible, cheaper for large datasets           │
│ └── Lake: Better for dedicated security/audit queries           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Integration with Other Services

```
┌─────────────────────────────────────────────────────────────────────┐
│           INTEGRATIONS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CloudWatch Logs:                                                     │
│ ├── Stream CloudTrail events to CloudWatch Logs                 │
│ ├── Create metric filters for security events                   │
│ ├── Set alarms on suspicious activity                           │
│ └── Example filter: Root login detection                        │
│     Filter: { $.userIdentity.type = "Root" }                    │
│     → Alarm: "Root account used!"                               │
│                                                                       │
│ EventBridge:                                                        │
│ ├── CloudTrail events automatically sent to EventBridge         │
│ ├── Create rules for specific API calls                         │
│ ├── Trigger Lambda, SNS, Step Functions on events              │
│ └── Example: Alert when security group modified                 │
│                                                                       │
│ Amazon Athena:                                                       │
│ ├── Query CloudTrail S3 logs with SQL                           │
│ ├── Create table from CloudTrail logs (auto-generated DDL)     │
│ ├── Partition by date for performance                           │
│ └── Cost-effective for ad-hoc historical analysis              │
│                                                                       │
│ AWS Config:                                                          │
│ ├── Config records resource configuration history              │
│ ├── CloudTrail records who changed it                          │
│ ├── Together: WHO changed WHAT and WHEN                        │
│ └── Compliance: "Who changed this security group?"            │
│                                                                       │
│ Security Hub:                                                        │
│ ├── Aggregates findings from CloudTrail                         │
│ ├── CIS benchmark checks using CloudTrail data                 │
│ └── Centralized security view                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "management-trail"
  s3_bucket_name                = aws_s3_bucket.trail.id
  include_global_service_events = true  # IAM, STS, etc.
  is_multi_region_trail         = true  # All regions
  is_organization_trail         = true  # All accounts
  enable_log_file_validation    = true
  kms_key_id                    = aws_kms_key.trail.arn

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.trail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.trail_cw.arn

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::sensitive-bucket/"]
    }
  }

  insight_selector {
    insight_type = "ApiCallRateInsight"
  }

  tags = { Environment = "prod" }
}
```

```bash
# Lookup events (last 90 days, free)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateInstances \
  --max-results 10

# Lookup by user
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=admin

# Get trail status
aws cloudtrail get-trail-status --name management-trail

# Validate log file integrity
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:123456:trail/management-trail \
  --start-time 2024-01-01T00:00:00Z
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Security Monitoring                                      │
│ CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS      │
│ ├── Root login: Filter {$.userIdentity.type = "Root"}           │
│ ├── Unauthorized calls: Filter {$.errorCode = "AccessDenied"}  │
│ ├── Console login without MFA                                     │
│ ├── IAM policy changes                                            │
│ └── Security group changes                                       │
│                                                                       │
│ Pattern 2: Compliance Audit Trail                                   │
│ ├── Organization trail (all accounts, all regions)              │
│ ├── S3 bucket with Object Lock (WORM — immutable)              │
│ ├── Log file validation enabled                                  │
│ ├── KMS encryption                                                │
│ ├── S3 Lifecycle: Move to Glacier after 90 days                 │
│ ├── Retain for 7 years (regulatory requirement)                 │
│ └── Athena for ad-hoc compliance queries                        │
│                                                                       │
│ Pattern 3: Incident Investigation                                   │
│ 1. Event History: Quick lookup of recent events                  │
│ 2. CloudTrail Lake: SQL queries for specific patterns           │
│ 3. Athena: Deep analysis of historical logs                      │
│ 4. Correlate with: VPC Flow Logs, GuardDuty findings           │
│                                                                       │
│ Pattern 4: Real-Time Response                                       │
│ CloudTrail → EventBridge → Lambda (auto-remediate)              │
│ ├── S3 bucket made public → Lambda reverts to private          │
│ ├── Security group opened 0.0.0.0/0 → Lambda removes rule    │
│ └── IAM user created → Lambda notifies security team           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
CloudTrail Quick Reference:
├── What: API audit log (who did what, when, where)
├── Event History: Free, 90 days, no setup needed
├── Trail: Delivers events to S3 (+ CW Logs, EventBridge)
├── Management events: Control plane (Create/Delete/Update)
├── Data events: Data plane (GetObject, Invoke) — extra cost
├── Insights: ML anomaly detection on API patterns
├── Organization trail: All accounts in one trail
├── Log validation: Detect tampering (always enable)
├── CloudTrail Lake: SQL queries on events
├── Key integrations: CloudWatch, EventBridge, Athena, Config
├── ⚡ Always enable multi-region + log validation
└── ⚡ Organization trail for multi-account
```

---

## What's Next?

In **Chapter 38: X-Ray**, we'll cover distributed tracing — tracing requests as they flow through your microservices architecture for debugging and performance analysis.
