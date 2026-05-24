# Chapter 3: Resource Hierarchy & Management (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: How Resources Are Identified in AWS](#part-1-how-resources-are-identified-in-aws)
- [Part 2: Resource Naming Conventions](#part-2-resource-naming-conventions)
- [Part 3: Tagging Strategy (Critical for Companies)](#part-3-tagging-strategy-critical-for-companies)
- [Part 4: Tag Policies (Organization-Level Enforcement)](#part-4-tag-policies-organization-level-enforcement)
- [Part 5: AWS Resource Groups](#part-5-aws-resource-groups)
- [Part 6: AWS Resource Explorer](#part-6-aws-resource-explorer)

---

## Overview

### Why Do Resource Organization and Tagging Matter?

> **Real-World Analogy:** Imagine a library. Every book (resource) has a unique ISBN (ARN). Books have labels on the spine (tags) — genre, author, shelf number. Librarians use these labels to find, organize, and inventory books. Without labels, finding anything in a library of 10,000 books is chaos. AWS tags are your labeling system for cloud resources.

In a real company with 500+ AWS resources, you **cannot** manage costs, security, or operations without a tagging strategy. Untagged resources are invisible to billing — you'll never know which team is burning $3,000/month. Tags are also how automation decides what to back up, what to shut down at night, and what needs patching.

This chapter covers how resources are organized, identified, and managed within AWS accounts — including ARNs, tagging strategies, Resource Groups, AWS Config, and real-world resource management patterns.

```
What you'll learn:
├── How resources are identified (ARNs)
├── Resource naming conventions
├── Tagging strategy (critical for companies)
├── AWS Resource Groups
├── AWS Resource Explorer
├── Tag Policies (Organization-level enforcement)
├── Service Quotas (limits)
├── AWS Config (compliance tracking)
├── Resource lifecycle management
└── Real-world tagging & management patterns
```

---

## Part 1: How Resources Are Identified in AWS

### Amazon Resource Names (ARNs)

Every resource in AWS has a unique identifier called an **ARN** (Amazon Resource Name).

```
ARN Format:
━━━━━━━━━━━
arn:partition:service:region:account-id:resource-type/resource-id

Parts explained:
┌─────────────────────────────────────────────────────────────────┐
│ Part         │ Description                │ Example             │
├─────────────────────────────────────────────────────────────────┤
│ arn          │ Fixed prefix               │ arn                 │
│ partition    │ AWS partition              │ aws (standard)      │
│              │                            │ aws-cn (China)      │
│              │                            │ aws-us-gov (GovCloud)│
│ service      │ AWS service namespace      │ s3, ec2, iam, rds   │
│ region       │ AWS region (or empty)      │ us-east-1, ap-south-1│
│ account-id   │ 12-digit AWS account ID   │ 123456789012        │
│ resource     │ Resource type + identifier │ instance/i-1234abcd │
└─────────────────────────────────────────────────────────────────┘
```

### ARN Examples

```
Common ARN patterns:

EC2 Instance:
  arn:aws:ec2:ap-south-1:123456789012:instance/i-0abcd1234efgh5678

S3 Bucket:
  arn:aws:s3:::my-bucket-name
  (No region, no account — S3 bucket names are globally unique)

S3 Object:
  arn:aws:s3:::my-bucket-name/folder/file.txt

IAM User:
  arn:aws:iam::123456789012:user/john.doe
  (No region — IAM is global)

IAM Role:
  arn:aws:iam::123456789012:role/MyAppRole

RDS Instance:
  arn:aws:rds:us-east-1:123456789012:db:my-database

Lambda Function:
  arn:aws:lambda:ap-south-1:123456789012:function:my-function

ECS Cluster:
  arn:aws:ecs:ap-south-1:123456789012:cluster/my-cluster

VPC:
  arn:aws:ec2:ap-south-1:123456789012:vpc/vpc-0abc123def456

Security Group:
  arn:aws:ec2:ap-south-1:123456789012:security-group/sg-0abc123

Load Balancer:
  arn:aws:elasticloadbalancing:ap-south-1:123456789012:loadbalancer/app/my-alb/50dc6c495c0c9188

CloudFormation Stack:
  arn:aws:cloudformation:us-east-1:123456789012:stack/my-stack/guid
```

### Where You'll See ARNs

```
ARNs are used in:
├── IAM Policies (Resource field)
│   "Resource": "arn:aws:s3:::my-bucket/*"
│
├── CLI commands (identifying resources)
│   aws ecs describe-services --cluster arn:aws:ecs:...
│
├── CloudFormation (referencing resources)
│   !GetAtt MyBucket.Arn
│
├── Event notifications (source identification)
│   "source": "arn:aws:s3:::my-bucket"
│
├── Cross-account access (trusting specific resources)
│   "Principal": {"AWS": "arn:aws:iam::999888777666:root"}
│
└── Tags, logs, billing (resource identification)
```

---

## Part 2: Resource Naming Conventions

### Why Naming Matters

```
Bad naming:                        Good naming:
├── server1                       ├── prod-web-app-01
├── test-db                       ├── prod-rds-postgres-users
├── my-bucket                     ├── techcorp-prod-assets-ap-south-1
├── function1                     ├── prod-api-order-processor
└── sg-default                    └── prod-sg-web-alb-ingress

Good names tell you:
├── Environment (prod/staging/dev)
├── Purpose (web, api, db, cache)
├── Application/Service name
├── Region (if multi-region)
└── Type of resource (optional)
```

### Naming Convention Template

```
Recommended pattern:
  <env>-<app/service>-<resource-type>-<identifier>

Examples:
┌─────────────────────────────────────────────────────────────────────┐
│ Resource Type    │ Convention                    │ Example            │
├─────────────────────────────────────────────────────────────────────┤
│ VPC              │ <env>-vpc-<purpose>           │ prod-vpc-main      │
│ Subnet           │ <env>-subnet-<type>-<az>      │ prod-subnet-pub-1a │
│ Security Group   │ <env>-sg-<app>-<purpose>      │ prod-sg-web-alb    │
│ EC2 Instance     │ <env>-<app>-<role>-<#>        │ prod-api-web-01    │
│ ALB              │ <env>-alb-<app>               │ prod-alb-frontend  │
│ RDS              │ <env>-rds-<engine>-<app>      │ prod-rds-pg-users  │
│ S3 Bucket        │ <company>-<env>-<purpose>-<region>│ tc-prod-logs-aps1│
│ Lambda           │ <env>-<app>-<function>        │ prod-orders-process│
│ ECS Cluster      │ <env>-ecs-<app>              │ prod-ecs-backend   │
│ ECR Repo         │ <app>/<service>              │ backend/user-svc   │
│ CloudWatch Alarm │ <env>-alarm-<resource>-<metric>│ prod-alarm-rds-cpu│
│ IAM Role         │ <env>-role-<purpose>          │ prod-role-ecs-task │
│ KMS Key          │ <env>-key-<purpose>          │ prod-key-rds-encrypt│
│ SNS Topic        │ <env>-topic-<purpose>        │ prod-topic-alerts  │
│ SQS Queue        │ <env>-queue-<purpose>        │ prod-queue-orders  │
└─────────────────────────────────────────────────────────────────────┘
```

### Naming Constraints

```
⚠️ Important constraints per service:

S3 Buckets:
├── Globally unique across ALL AWS accounts
├── 3-63 characters
├── Lowercase letters, numbers, hyphens only
├── Must start with letter or number
└── No periods (avoid for SSL compatibility)

EC2 (Name tag):
├── Up to 256 characters
├── Any characters allowed
└── Not technically a "name" — it's just the Name tag

IAM Resources:
├── Alphanumeric + these: +=,.@-_
├── User/Role names: 1-64 characters
├── Path-based organization: /app/prod/role-name

RDS:
├── 1-63 characters
├── Alphanumeric + hyphens
├── Must start with letter
├── No consecutive hyphens, can't end with hyphen
└── Lowercase only

Lambda:
├── 1-64 characters
├── Letters, numbers, hyphens, underscores
└── Case-sensitive
```

---

## Part 3: Tagging Strategy (Critical for Companies)

### What are Tags?

Tags are **key-value pairs** attached to AWS resources for identification, organization, cost allocation, automation, and access control.

```
Tag structure:
┌─────────────────────────────────────────────┐
│ Key              │ Value                     │
├─────────────────────────────────────────────┤
│ Environment      │ production                │
│ Team             │ backend                   │
│ Application      │ order-service             │
│ CostCenter       │ CC-1234                   │
│ Owner            │ john.doe@company.com      │
│ ManagedBy        │ terraform                 │
│ CreatedDate      │ 2026-05-16                │
│ DataClassification│ confidential             │
└─────────────────────────────────────────────┘
```

### Tag Categories

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TAG CATEGORIES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. BUSINESS TAGS (Cost allocation & ownership)                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ CostCenter        │ CC-1234, CC-5678 │ Billing chargeback    │    │
│  │ Project           │ project-name     │ Project association    │    │
│  │ Owner             │ email            │ Who's responsible      │    │
│  │ Department        │ engineering      │ Department tracking    │    │
│  │ BusinessUnit      │ retail, finance  │ BU cost attribution    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  2. TECHNICAL TAGS (Operations & automation)                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ Environment       │ prod/staging/dev │ Environment isolation  │    │
│  │ Application       │ app-name         │ Application grouping   │    │
│  │ Component         │ api/web/worker   │ Component within app   │    │
│  │ ManagedBy         │ terraform/cdk    │ IaC tool tracking      │    │
│  │ Version           │ v1.2.3           │ Deployment version     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  3. AUTOMATION TAGS (Trigger automated actions)                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ AutoShutdown      │ true/false       │ Stop instances at night│    │
│  │ BackupSchedule    │ daily/weekly     │ Automated backups      │    │
│  │ PatchGroup        │ group-a/group-b  │ Patch management       │    │
│  │ AutoScale         │ true             │ Include in ASG         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  4. SECURITY TAGS (Access control & compliance)                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ DataClassification│ public/internal/ │ Data sensitivity       │    │
│  │                   │ confidential     │                        │    │
│  │ Compliance        │ pci/hipaa/gdpr   │ Regulatory compliance  │    │
│  │ SecurityZone      │ dmz/internal     │ Network zone           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Tagging Best Practices

```
Rules for effective tagging:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. MANDATORY TAGS (enforce via Tag Policies / AWS Config):
   ├── Environment
   ├── Application (or Project)
   ├── Owner
   └── CostCenter

2. CONSISTENT CASING:
   ├── Keys: PascalCase (Environment, CostCenter)
   │   or lowercase with hyphens (environment, cost-center)
   └── Values: lowercase (production, staging, dev)
   
   ⚠️ Tags are CASE SENSITIVE!
   "Environment" ≠ "environment" ≠ "ENVIRONMENT"

3. TAG LIMITS:
   ├── Max 50 tags per resource
   ├── Key: max 128 characters
   ├── Value: max 256 characters
   └── Keys cannot start with "aws:" (reserved)

4. NOT ALL RESOURCES SUPPORT TAGS:
   Some resources that DON'T support tags:
   ├── Individual IAM policies
   ├── CloudWatch log events
   ├── Route table entries
   └── Security group rules (the SG itself supports tags)

5. TAG PROPAGATION:
   Some services propagate tags:
   ├── Auto Scaling → propagates tags to launched instances
   ├── CloudFormation → can propagate stack tags to resources
   └── ECS → task-level tags (not always to underlying EC2)
```

### How to Apply Tags

```
Console: Any resource → Tags tab → Add tag

CLI:
  # Tag an EC2 instance
  aws ec2 create-tags \
    --resources i-0abc123def456 \
    --tags Key=Environment,Value=production Key=Team,Value=backend

  # Tag an S3 bucket
  aws s3api put-bucket-tagging \
    --bucket my-bucket \
    --tagging 'TagSet=[{Key=Environment,Value=production}]'

  # Tag an RDS instance
  aws rds add-tags-to-resource \
    --resource-name arn:aws:rds:ap-south-1:123456789012:db:my-db \
    --tags Key=Environment,Value=production

Terraform:
  resource "aws_instance" "web" {
    # ... instance config ...
    
    tags = {
      Name        = "prod-web-01"
      Environment = "production"
      Application = "frontend"
      Team        = "platform"
      ManagedBy   = "terraform"
      CostCenter  = "CC-1234"
    }
  }

CloudFormation:
  Resources:
    MyInstance:
      Type: AWS::EC2::Instance
      Properties:
        # ... config ...
        Tags:
          - Key: Environment
            Value: production
          - Key: Application
            Value: frontend
```

---

## Part 4: Tag Policies (Organization-Level Enforcement)

### What are Tag Policies?

Tag Policies standardize tags across your AWS Organization — ensuring consistent key names and allowed values.

```
Console → AWS Organizations → Policies → Tag policies → Create

┌─────────────────────────────────────────────────────────────────┐
│                     TAG POLICY EXAMPLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Policy name: "Standard Tags"                                     │
│                                                                   │
│ This policy ensures:                                             │
│ ├── "Environment" tag uses exact casing                         │
│ ├── Values limited to: production, staging, development         │
│ ├── Enforced on: EC2, RDS, S3                                  │
│ └── Non-compliant resources flagged                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```json
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": [
          "production",
          "staging",
          "development",
          "sandbox"
        ]
      },
      "enforced_for": {
        "@@assign": [
          "ec2:instance",
          "rds:db",
          "s3:bucket",
          "elasticloadbalancing:loadbalancer"
        ]
      }
    },
    "CostCenter": {
      "tag_key": {
        "@@assign": "CostCenter"
      },
      "tag_value": {
        "@@assign": [
          "CC-1001",
          "CC-1002",
          "CC-2001",
          "CC-3001"
        ]
      }
    }
  }
}
```

```
⚠️ Important: Tag Policies DON'T prevent resource creation!
   They only report non-compliance.
   
   To ENFORCE tags (deny creation without tags):
   Use AWS Config rules + remediation
   or SCP with condition on tag presence
   or IAM policies with tag conditions
```

---

## Part 5: AWS Resource Groups

### What are Resource Groups?

Resource Groups let you organize resources that share common tags or belong to the same CloudFormation stack.

```
Console → Resource Groups & Tag Editor → Create resource group

┌─────────────────────────────────────────────────────────────────┐
│                    CREATE RESOURCE GROUP                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Group type:                                                      │
│ ○ Tag-based (resources matching tag criteria)                   │
│ ○ CloudFormation stack-based (all resources in a stack)         │
│                                                                   │
│ Group name: [prod-frontend-resources]                            │
│                                                                   │
│ Tag-based criteria:                                              │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Resource types: [All supported types] or specific          │   │
│ │                                                            │   │
│ │ Tags:                                                      │   │
│ │ ├── Key: Environment    Value: production                  │   │
│ │ └── Key: Application    Value: frontend                    │   │
│ │                                                            │   │
│ │ (Resources matching ALL criteria will be in this group)    │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create group]                                                   │
│                                                                   │
│ 💡 Use cases:                                                    │
│ ├── View all resources for an application together              │
│ ├── Run Automation (Systems Manager) on grouped resources       │
│ ├── Apply CloudWatch insights to a group                        │
│ ├── View costs for grouped resources                            │
│ └── Execute operations (start/stop) on all grouped resources   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Resource Groups vs Azure Resource Groups

```
⚠️ IMPORTANT DISTINCTION:

Azure Resource Groups:
├── MANDATORY — every resource must be in exactly one RG
├── Lifecycle bound — delete RG = delete all resources
├── Permission boundary — RBAC applies at RG level
├── Deployment target — deploy templates to a RG
└── Physical container

AWS Resource Groups:
├── OPTIONAL — resources exist independently of groups
├── Just a VIEW — deleting group doesn't delete resources
├── No permission boundary — just organizational
├── Query-based — dynamically includes matching resources
└── Logical grouping only

In AWS, the real "container" is the ACCOUNT itself.
Resource Groups are just a convenience for viewing/managing.
```

---

## Part 6: AWS Resource Explorer

### What is Resource Explorer?

Resource Explorer provides a single search experience to find resources across all regions in your account.

```
Console → Resource Explorer

┌─────────────────────────────────────────────────────────────────┐
│                    RESOURCE EXPLORER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Search: [my-application] or [i-0abc123] or [tag:Env=prod]       │
│                                                                   │
│ Results:                                                         │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Resource         │ Type      │ Region      │ Tags          │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ prod-web-01     │ EC2       │ ap-south-1  │ Env:prod      │   │
│ │ prod-web-02     │ EC2       │ ap-south-1  │ Env:prod      │   │
│ │ prod-api-alb    │ ALB       │ ap-south-1  │ Env:prod      │   │
│ │ prod-rds-main   │ RDS       │ ap-south-1  │ Env:prod      │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Features:                                                        │
│ ├── Search across ALL regions (no switching regions!)           │
│ ├── Search by tag, name, type, ARN                             │
│ ├── Create views (saved searches)                               │
│ ├── Aggregator index (one region indexes all)                   │
│ └── Free to use                                                  │
│                                                                   │
│ Setup:                                                           │
│ Console → Resource Explorer → Turn on                           │
│ Select aggregator region (where index lives)                    │
│ Wait ~15 min for initial index                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Tag Editor (Bulk Tagging)

```
Console → Resource Groups & Tag Editor → Tag Editor

┌─────────────────────────────────────────────────────────────────┐
│                        TAG EDITOR                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Find resources to tag:                                           │
│ ├── Regions: [All regions] or specific                          │
│ ├── Resource types: [EC2:Instance, RDS:DB, S3:Bucket...]        │
│ └── Tags (filter): Key=Environment, Value=production            │
│                                                                   │
│ [Search resources]                                               │
│                                                                   │
│ Results: 47 resources found                                      │
│ [✓ Select all] or select specific resources                     │
│                                                                   │
│ Manage tags of selected resources:                               │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Action          │ Key          │ Value                     │   │
│ ├───────────────────────────────────────────────────────────┤   │
│ │ Add/Overwrite   │ CostCenter   │ CC-1234                  │   │
│ │ Add/Overwrite   │ Team         │ platform                 │   │
│ │ Remove          │ OldTag       │                          │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Apply changes to all selected resources]                        │
│                                                                   │
│ 💡 Use for: Bulk-adding missing tags to existing resources       │
│ 💡 Use for: Correcting tag values across many resources          │
│ 💡 Use for: Removing deprecated tags                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Service Quotas (Resource Limits)

### What are Service Quotas?

Every AWS account has default limits on resources. Understanding and managing these is critical.

```
Console → Service Quotas

┌─────────────────────────────────────────────────────────────────┐
│                   COMMON SERVICE QUOTAS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ EC2:                                                             │
│ ├── Running On-Demand instances: varies by type                 │
│ │   (e.g., 5 per instance type for new accounts)               │
│ ├── Elastic IPs: 5 per region                                  │
│ ├── Security Groups per VPC: 2,500                             │
│ └── Rules per Security Group: 60 inbound + 60 outbound         │
│                                                                   │
│ VPC:                                                             │
│ ├── VPCs per region: 5 (can request increase)                   │
│ ├── Subnets per VPC: 200                                       │
│ ├── Internet Gateways per region: 5                            │
│ ├── NAT Gateways per AZ: 5                                    │
│ └── Route tables per VPC: 200                                  │
│                                                                   │
│ S3:                                                              │
│ ├── Buckets per account: 100 (can request 1000)                │
│ ├── Object size: 5 TB max                                      │
│ └── No limit on total storage                                  │
│                                                                   │
│ RDS:                                                             │
│ ├── DB instances per region: 40                                 │
│ ├── Storage: 64 TB (varies by engine)                          │
│ └── Read replicas per primary: 5 (Aurora: 15)                  │
│                                                                   │
│ Lambda:                                                          │
│ ├── Concurrent executions: 1,000 (can increase)                │
│ ├── Function memory: 128 MB - 10 GB                            │
│ ├── Timeout: 15 minutes max                                    │
│ └── Package size: 250 MB (unzipped)                            │
│                                                                   │
│ IAM:                                                             │
│ ├── Users per account: 5,000                                   │
│ ├── Roles per account: 1,000                                   │
│ ├── Groups per account: 300                                    │
│ └── Policies per account: 1,500                                │
│                                                                   │
│ ECS/EKS:                                                         │
│ ├── Clusters per region: 10,000                                │
│ ├── Services per cluster: 5,000                                │
│ └── Tasks per service: varies                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Requesting Quota Increases

```
Console → Service Quotas → Select service → Select quota → Request increase

┌─────────────────────────────────────────────────────────────────┐
│ Service: Amazon EC2                                              │
│ Quota: Running On-Demand Standard instances                     │
│ Current value: 5                                                │
│ Requested value: [20]                                           │
│                                                                   │
│ [Submit]                                                         │
│                                                                   │
│ ⏱️ Processing time: Usually 1-3 business days                    │
│ 💡 Some quotas are auto-approved instantly                       │
│ 💡 Request BEFORE you need it (don't wait for errors)           │
│                                                                   │
│ CLI:                                                             │
│ aws service-quotas request-service-quota-increase \              │
│   --service-code ec2 \                                          │
│   --quota-code L-1216C47A \                                     │
│   --desired-value 20                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 9: AWS Config (Resource Compliance)

### What is AWS Config?

AWS Config continuously monitors and records your resource configurations, and evaluates them against desired configurations (rules).

```
┌─────────────────────────────────────────────────────────────────┐
│                       AWS CONFIG                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  What it does:                                                   │
│  ├── Records resource configuration changes over time           │
│  ├── Evaluates configurations against compliance rules          │
│  ├── Provides configuration timeline (history)                  │
│  ├── Shows relationships between resources                      │
│  └── Can auto-remediate non-compliant resources                 │
│                                                                   │
│  Use cases:                                                      │
│  ├── "Is S3 bucket encryption enabled?" → Rule checks           │
│  ├── "When did this SG rule change?" → Configuration timeline  │
│  ├── "Which resources are non-compliant?" → Dashboard          │
│  ├── "Auto-fix untagged resources" → Remediation action        │
│  └── "Who changed this?" → Paired with CloudTrail              │
│                                                                   │
│  Flow:                                                           │
│  Resource Change → Config Records → Rule Evaluates →            │
│  ├── COMPLIANT (all good)                                       │
│  └── NON_COMPLIANT → Alert / Remediate                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Common AWS Config Rules

```
Console → AWS Config → Rules → Add rule

Popular managed rules:
┌──────────────────────────────────────────────────────────────────┐
│ Rule Name                          │ What it checks              │
├──────────────────────────────────────────────────────────────────┤
│ required-tags                      │ Resources have required tags│
│ s3-bucket-server-side-encryption   │ S3 encryption enabled       │
│ s3-bucket-public-read-prohibited   │ No public S3 buckets        │
│ ec2-instance-no-public-ip          │ No public IPs on EC2        │
│ rds-instance-public-access-check   │ RDS not publicly accessible │
│ encrypted-volumes                  │ EBS volumes encrypted       │
│ vpc-flow-logs-enabled              │ VPC flow logs turned on     │
│ cloudtrail-enabled                 │ CloudTrail active           │
│ iam-user-mfa-enabled              │ Users have MFA              │
│ root-account-mfa-enabled          │ Root has MFA                │
│ restricted-ssh                     │ No 0.0.0.0/0 on port 22    │
│ rds-multi-az-support              │ RDS has Multi-AZ            │
│ ebs-optimized-instance            │ EBS-optimized EC2           │
│ approved-amis-by-tag              │ Only approved AMIs used     │
└──────────────────────────────────────────────────────────────────┘
```

### Config Remediation

```
Auto-fix non-compliant resources:

Rule: "required-tags" (check for Environment tag)
↓
Resource missing tag → NON_COMPLIANT
↓
Remediation Action: AWS Systems Manager Automation document
  → Adds default tag: Environment=unknown
↓
Resource now COMPLIANT (or flagged for human review)

Setup:
Console → Config → Rules → Select rule → Manage remediation
├── Automatic remediation: Fix immediately when detected
└── Manual remediation: Require human click to fix
```

---

## Part 10: AWS Resource Hierarchy - Complete Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│              COMPLETE AWS RESOURCE HIERARCHY                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  AWS Organization                                                    │
│  │                                                                   │
│  ├── Root                                                           │
│  │   ├── SCPs (guardrails)                                         │
│  │   └── Tag Policies (standardization)                            │
│  │                                                                   │
│  ├── Organizational Unit                                            │
│  │   ├── SCPs (inherited + own)                                    │
│  │   └── Tag Policies (inherited + own)                            │
│  │                                                                   │
│  └── AWS Account (isolation boundary)                               │
│      │                                                               │
│      ├── IAM (users, roles, policies — account-level)              │
│      │                                                               │
│      ├── Region: ap-south-1                                        │
│      │   │                                                           │
│      │   ├── VPC (network isolation)                                │
│      │   │   ├── Subnet (public/private)                           │
│      │   │   │   └── EC2, RDS, ECS tasks, Lambda ENIs             │
│      │   │   ├── Security Groups (firewall)                        │
│      │   │   ├── Route Tables                                      │
│      │   │   └── NACLs                                             │
│      │   │                                                           │
│      │   ├── S3 Buckets (region-level)                             │
│      │   ├── RDS Instances                                         │
│      │   ├── ECS Clusters                                          │
│      │   ├── Lambda Functions                                      │
│      │   └── ... all regional resources                            │
│      │                                                               │
│      ├── Region: us-east-1                                         │
│      │   └── ... resources in this region                          │
│      │                                                               │
│      └── Global Resources (not region-specific):                    │
│          ├── IAM (users, roles, policies)                          │
│          ├── Route 53 (DNS)                                        │
│          ├── CloudFront (CDN)                                      │
│          ├── WAF (global)                                          │
│          └── Organizations                                         │
│                                                                       │
│  TAGGING spans across everything:                                    │
│  Any resource that supports tags can be tagged,                     │
│  regardless of where it sits in the hierarchy.                      │
│  Tags are the PRIMARY mechanism for logical organization.           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Real-World Resource Management Patterns

### Pattern 1: Tag-Based Cost Allocation

```
Company requirement: "Show me spending by team and environment"

Solution:
1. Define mandatory tags:
   ├── Team: frontend, backend, data, platform
   └── Environment: production, staging, development

2. Enforce via AWS Config rule: required-tags
   - Auto-remediation: Tag untagged resources with "unknown"

3. Activate cost allocation tags:
   Console → Billing → Cost allocation tags → Activate
   
4. Use Cost Explorer:
   Group by: Tag (Team) → Shows cost per team
   Filter by: Tag (Environment=production) → Prod costs only

5. Create monthly cost reports per team → email to team leads
```

### Pattern 2: Automation with Tags

```
Scenario: Stop dev EC2 instances at night to save money

Tag: AutoShutdown=true + Schedule=nights-weekends

Lambda function (triggered by EventBridge at 8 PM):
  1. Query EC2 instances with tag AutoShutdown=true
  2. Stop all matching instances
  3. Another trigger at 8 AM → Start them back

Lambda function (rough logic):
  instances = ec2.describe_instances(
    Filters=[{'Name': 'tag:AutoShutdown', 'Values': ['true']}]
  )
  ec2.stop_instances(InstanceIds=[...])

Savings: ~65% on dev EC2 costs (14 hours stopped per day)
```

### Pattern 3: Resource Lifecycle Management

```
Track resource lifecycle with tags:

CreatedDate: 2026-05-16
CreatedBy: john.doe@company.com (via CloudTrail)
ExpiresOn: 2026-08-16 (for temporary resources)
Purpose: "Load testing for Q3 release"

Automated cleanup Lambda:
  1. Find resources with ExpiresOn tag in the past
  2. Send notification: "These resources expire today"
  3. After 7 days past expiry → Auto-terminate
  4. Log everything to CloudTrail
  
Prevents "zombie resources" that nobody owns.
```

---

## Quick Reference: Console Navigation

| Task | Console Path |
|------|-------------|
| View resource ARN | Any resource → Details page → ARN field |
| Tag a resource | Any resource → Tags tab → Manage tags |
| Bulk tag resources | Resource Groups → Tag Editor |
| Create Resource Group | Resource Groups → Create group |
| Resource Explorer | Resource Explorer → Search |
| Service Quotas | Service Quotas → Select service |
| AWS Config | AWS Config → Dashboard / Rules |
| Tag Policies | Organizations → Policies → Tag policies |
| Cost allocation tags | Billing → Cost allocation tags |
| Cost Explorer (by tag) | Cost Explorer → Group by → Tag |

---

## What's Next?

In the next chapter, we'll cover IAM (Identity & Access Management) in depth — users, groups, roles, policies, federation, and how to implement least-privilege access.

→ Next: [Chapter 4: IAM - Identity & Access Management](04-iam.md)

---

*Last Updated: May 2026*
