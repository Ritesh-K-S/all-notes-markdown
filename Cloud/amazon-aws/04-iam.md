# Chapter 4: IAM - Identity & Access Management (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: IAM Fundamentals](#part-1-iam-fundamentals)
- [Part 2: IAM Users](#part-2-iam-users)
- [Part 3: IAM Groups](#part-3-iam-groups)
- [Part 4: IAM Policies (The Core of Authorization)](#part-4-iam-policies-the-core-of-authorization)
- [Part 5: IAM Roles](#part-5-iam-roles)
- [Part 6: MFA (Multi-Factor Authentication)](#part-6-mfa-multi-factor-authentication)

---

## Overview

### Why is IAM the Most Important AWS Service?

> **Real-World Analogy:** IAM is like a corporate building's security system. **Authentication** is the ID badge scan at the front door ("prove you work here"). **Authorization** is the keycard permissions ("you can enter floors 1-3 but not the server room"). **Policies** are the access rules programmed into the system. **Roles** are temporary visitor badges — "here's a day pass to the lab, give it back tonight."

IAM is the **#1 source of cloud security breaches**. A misconfigured policy or leaked access key has caused companies to lose millions. Every AWS service depends on IAM permissions. If you don't understand IAM, you'll either lock yourself out or leave the door wide open.

AWS IAM is the service that controls **who** (authentication) can do **what** (authorization) on **which resources** in your AWS account. It's free, global, and the foundation of all AWS security.

```
What you'll learn:
├── IAM fundamentals (how authentication & authorization work)
├── IAM Users (creation, console vs programmatic access)
├── IAM Groups (organizing users)
├── IAM Policies (JSON structure, types, evaluation logic)
├── IAM Roles (for services, cross-account, federation)
├── MFA (Multi-Factor Authentication)
├── Access Keys & Secrets (programmatic access)
├── Permission Boundaries (limiting max permissions)
├── AWS Identity Center (SSO) — the modern way
├── Service-linked Roles & Service Control Policies
├── Policy evaluation logic (how AWS decides allow/deny)
├── Security best practices
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: IAM Fundamentals

### How Authentication & Authorization Work

```
┌─────────────────────────────────────────────────────────────────────┐
│                   IAM: Authentication + Authorization                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  REQUEST FLOW:                                                       │
│                                                                       │
│  User/Service ──→ AWS Endpoint ──→ Authentication ──→ Authorization  │
│       │                                    │                │         │
│       │                                    ▼                ▼         │
│       │                            "Who are you?"   "Can you do this?"│
│       │                                    │                │         │
│       │                                    ▼                ▼         │
│       │                            Credentials       Policy Check    │
│       │                            verified          (Allow/Deny)    │
│       │                                                              │
│  Authentication methods:           Authorization checks:             │
│  ├── Console: Username + Password  ├── Identity-based policies       │
│  ├── CLI/SDK: Access Key + Secret  ├── Resource-based policies       │
│  ├── Role: Temporary credentials   ├── Permission boundaries         │
│  └── Federation: External IdP      ├── Organizations SCPs            │
│                                     ├── Session policies             │
│                                     └── ACLs (legacy)                │
│                                                                       │
│  DEFAULT: Everything is DENIED                                       │
│  Must have explicit ALLOW, and no explicit DENY                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### IAM is Global

```
⚠️ IAM is a GLOBAL service:
├── Users, Groups, Roles, Policies → NOT region-specific
├── Created once, available in ALL regions
├── IAM console URL: console.aws.amazon.com/iam/
└── Changes propagate globally (eventual consistency, usually seconds)

Root Account:
├── Created when you first create AWS account
├── Has FULL unrestricted access to everything
├── ⚠️ NEVER use for daily tasks
├── ⚠️ NEVER create access keys for root
├── ✅ Enable MFA immediately
├── ✅ Use only for: account settings, billing, closing account
└── ✅ Lock away root credentials
```

---

## Part 2: IAM Users

### What is an IAM User?

An IAM User represents a **person or application** that interacts with AWS.

```
Portal → IAM → Users → Create user

┌─────────────────────────────────────────────────────────────────┐
│                      CREATE IAM USER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: User details                                             │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ User name:  [john.doe]                                    │   │
│ │             Rules: 1-64 chars, alphanumeric + +=,.@-_     │   │
│ │                                                            │   │
│ │ ☐ Provide user access to the AWS Management Console       │   │
│ │   (Check this for human users who need Console)           │   │
│ │                                                            │   │
│ │ If checked:                                                │   │
│ │   ○ User must create password at next sign-in (recommended)│   │
│ │   ○ Custom password: [**********]                          │   │
│ │                                                            │   │
│ │ ☑ User must create new password at next sign-in           │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 2: Set permissions                                          │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ○ Add user to group (RECOMMENDED)                         │   │
│ │   → Select existing group or create new                   │   │
│ │                                                            │   │
│ │ ○ Copy permissions from existing user                      │   │
│ │                                                            │   │
│ │ ○ Attach policies directly                                │   │
│ │   ⚠️ Not recommended (hard to manage at scale)            │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 3: Review and create                                        │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Tags (optional):                                           │   │
│ │   Department: Engineering                                  │   │
│ │   Team: Backend                                           │   │
│ │   Title: Senior Developer                                 │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create user]                                                    │
│                                                                   │
│ ⚠️ Download or copy the sign-in URL and credentials!             │
│    Sign-in URL: https://123456789012.signin.aws.amazon.com/console│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create user
  aws iam create-user --user-name john.doe --tags Key=Team,Value=Backend

  # Create console password
  aws iam create-login-profile --user-name john.doe \
    --password "TempP@ss123!" --password-reset-required

  # Create access keys (programmatic access)
  aws iam create-access-key --user-name john.doe
  # Returns: AccessKeyId + SecretAccessKey (show only ONCE!)
```

### IAM User Limits

```
Per AWS account:
├── Max IAM users: 5,000
├── Max groups per user: 10
├── Max access keys per user: 2 (for rotation)
├── Max MFA devices per user: 8
├── Max inline policies per user: will count toward 2,048 char limit
└── Max managed policies per user: 10
```

---

## Part 3: IAM Groups

### What is an IAM Group?

A Group is a **collection of IAM users**. Policies attached to the group apply to all members.

```
Portal → IAM → User groups → Create group

┌─────────────────────────────────────────────────────────────────┐
│                    CREATE IAM GROUP                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Group name: [Developers]                                         │
│                                                                   │
│ Add users to group:                                              │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ☑ john.doe                                                │   │
│ │ ☑ jane.smith                                              │   │
│ │ ☑ bob.wilson                                              │   │
│ │ ☐ alice.admin (she's in AdminGroup)                       │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Attach permissions policies:                                     │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ☑ AmazonEC2FullAccess                                     │   │
│ │ ☑ AmazonS3FullAccess                                      │   │
│ │ ☑ AmazonRDSReadOnlyAccess                                 │   │
│ │ ☑ CloudWatchLogsFullAccess                                 │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create group]                                                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Key rules:
├── A group is NOT an identity (cannot be used as Principal in resource policies)
├── Groups cannot be nested (no group inside group)
├── A user can belong to max 10 groups
├── Max 300 groups per account
├── Max 10 managed policies per group
└── ✅ ALWAYS assign permissions via groups, not directly to users

CLI:
  aws iam create-group --group-name Developers
  aws iam add-user-to-group --group-name Developers --user-name john.doe
  aws iam attach-group-policy --group-name Developers \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

### Recommended Group Structure

```
┌─────────────────────────────────────────────────────────────────┐
│              RECOMMENDED IAM GROUPS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ By Role:                                                         │
│ ├── Admins          → AdministratorAccess                       │
│ ├── Developers      → EC2, S3, Lambda, DynamoDB, CloudWatch     │
│ ├── DevOps          → Full access to most services              │
│ ├── DataEngineers   → S3, Glue, Athena, Redshift, EMR          │
│ ├── QA              → EC2 (limited), S3 (read), CloudWatch      │
│ ├── Finance         → Billing read-only, Cost Explorer          │
│ ├── SecurityAudit   → SecurityAudit policy, read-only           │
│ └── ReadOnly        → ViewOnlyAccess (for stakeholders)         │
│                                                                   │
│ ⚠️ A user can be in multiple groups                              │
│    john.doe → Developers + ReadOnly                             │
│    Effective permissions = UNION of all group policies           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: IAM Policies (The Core of Authorization)

### What is an IAM Policy?

A Policy is a **JSON document** that defines permissions. It specifies what actions are allowed or denied on which resources.

```
┌─────────────────────────────────────────────────────────────────┐
│                   POLICY TYPES                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 1. AWS Managed Policies (created by AWS)                        │
│    ├── Examples: AmazonEC2FullAccess, AmazonS3ReadOnlyAccess   │
│    ├── ✅ Maintained by AWS (updated for new features)           │
│    ├── ⚠️ Often too broad for production                        │
│    └── ARN: arn:aws:iam::aws:policy/PolicyName                  │
│                                                                   │
│ 2. Customer Managed Policies (created by you)                   │
│    ├── Custom policies for your specific needs                  │
│    ├── ✅ Can be versioned (up to 5 versions)                    │
│    ├── ✅ Reusable (attach to multiple users/groups/roles)       │
│    └── ARN: arn:aws:iam::123456789012:policy/PolicyName         │
│                                                                   │
│ 3. Inline Policies (embedded in user/group/role)                │
│    ├── ⚠️ Not reusable, hard to manage                          │
│    ├── Used when policy should NOT be accidentally applied      │
│    │   to other entities                                        │
│    └── Deleted when user/group/role is deleted                  │
│                                                                   │
│ 4. Resource-Based Policies (on the resource itself)             │
│    ├── S3 Bucket Policies, SQS Queue Policies, KMS Key Policies│
│    ├── Specify WHO can access THIS resource                     │
│    └── Can grant cross-account access                           │
│                                                                   │
│ 5. Permission Boundaries (max permissions)                       │
│    ├── Sets maximum possible permissions for a user/role        │
│    ├── Even if identity policy grants access, boundary limits it│
│    └── Used for delegation (let devs create roles safely)       │
│                                                                   │
│ 6. Service Control Policies (SCPs — AWS Organizations)          │
│    ├── Limits maximum permissions for entire accounts           │
│    ├── Applied at OU or account level                           │
│    └── Does NOT grant permissions, only restricts               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Policy JSON Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2Describe",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-south-1"
        }
      }
    }
  ]
}
```

```
Policy elements explained:
┌─────────────────────────────────────────────────────────────────────┐
│ Element     │ Required │ Description                                 │
├─────────────────────────────────────────────────────────────────────┤
│ Version     │ Yes      │ Always "2012-10-17" (latest policy version) │
│ Statement   │ Yes      │ Array of permission statements              │
│ Sid         │ No       │ Statement ID (human-readable identifier)    │
│ Effect      │ Yes      │ "Allow" or "Deny"                           │
│ Action      │ Yes      │ API actions (service:action)                │
│ Resource    │ Yes*     │ ARN of resource(s) this applies to          │
│ Condition   │ No       │ When this statement applies                 │
│ Principal   │ Some     │ WHO (only in resource-based policies)       │
│ NotAction   │ No       │ Everything EXCEPT these actions             │
│ NotResource │ No       │ Everything EXCEPT these resources           │
└─────────────────────────────────────────────────────────────────────┘

* Resource can be "*" (all resources) for non-resource-level actions
```

### Common Policy Examples

```json
// Example 1: Allow S3 access to specific bucket only
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::company-prod-data"
    },
    {
      "Sid": "AllowObjectActions",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::company-prod-data/*"
    }
  ]
}

// Example 2: Allow EC2 actions only in specific region
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2InMumbai",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-south-1"
        }
      }
    }
  ]
}

// Example 3: Deny delete on production resources (tag-based)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteProduction",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "s3:DeleteBucket"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Environment": "production"
        }
      }
    }
  ]
}

// Example 4: Force MFA for sensitive actions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyWithoutMFA",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}

// Example 5: Self-manage IAM credentials (let users change own password)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowManageOwnCredentials",
      "Effect": "Allow",
      "Action": [
        "iam:ChangePassword",
        "iam:CreateAccessKey",
        "iam:DeleteAccessKey",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:EnableMFADevice",
        "iam:DeactivateMFADevice"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
```

### Action Format & Wildcards

```
Action format: <service>:<action>

Examples:
├── s3:GetObject          → Get single object
├── s3:Get*               → All Get actions (GetObject, GetBucketPolicy...)
├── s3:*                  → All S3 actions
├── ec2:Describe*         → All Describe actions
├── ec2:RunInstances      → Launch instances
├── rds:*                 → All RDS actions
├── *                     → ALL actions on ALL services (admin)
└── iam:Create*           → All IAM create actions

Common service prefixes:
├── ec2:        → Compute (VMs, VPC, Security Groups)
├── s3:         → Storage
├── rds:        → Databases
├── lambda:     → Serverless functions
├── iam:        → Identity management
├── sts:        → Security Token Service (assume role)
├── logs:       → CloudWatch Logs
├── cloudwatch: → CloudWatch Metrics & Alarms
├── dynamodb:   → DynamoDB
├── sqs:        → Queue service
├── sns:        → Notification service
├── kms:        → Key Management
├── secretsmanager: → Secrets Manager
└── ecs: / eks: → Container services
```

---

## Part 5: IAM Roles

### What is an IAM Role?

A Role is an identity with permissions that can be **assumed** by trusted entities (services, users, external accounts). Unlike users, roles have **no permanent credentials** — they provide temporary security credentials.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        IAM ROLES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Role = Trust Policy + Permission Policy                             │
│                                                                       │
│  ┌──────────────────────┐    ┌────────────────────────────────┐     │
│  │ Trust Policy          │    │ Permission Policy               │     │
│  │ (Who can assume)      │    │ (What they can do)              │     │
│  ├──────────────────────┤    ├────────────────────────────────┤     │
│  │ • EC2 service         │    │ • s3:GetObject                  │     │
│  │ • Lambda service      │    │ • s3:PutObject                  │     │
│  │ • Account 9876543210  │    │ • dynamodb:Query                │     │
│  │ • SAML provider       │    │ • logs:PutLogEvents             │     │
│  │ • Cognito users       │    │                                 │     │
│  └──────────────────────┘    └────────────────────────────────┘     │
│                                                                       │
│  When assumed: Returns temporary credentials                         │
│  ├── AccessKeyId (temporary)                                        │
│  ├── SecretAccessKey (temporary)                                    │
│  ├── SessionToken (required for temp creds)                         │
│  └── Expiration (default: 1 hour, max: 12 hours)                   │
│                                                                       │
│  Common use cases:                                                   │
│  ├── EC2 instance accessing S3 (instance profile)                   │
│  ├── Lambda function accessing DynamoDB                              │
│  ├── Cross-account access (Account A → Account B)                   │
│  ├── Federation (SSO users from corporate AD)                       │
│  └── ECS/EKS tasks/pods accessing AWS services                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Role (for EC2 Service)

```
Portal → IAM → Roles → Create role

┌─────────────────────────────────────────────────────────────────┐
│                     CREATE ROLE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Select trusted entity                                    │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ○ AWS service (EC2, Lambda, ECS...)                       │   │
│ │ ○ AWS account (cross-account access)                      │   │
│ │ ○ Web identity (Cognito, OIDC provider)                   │   │
│ │ ○ SAML 2.0 federation (corporate SSO)                     │   │
│ │ ○ Custom trust policy (write JSON)                        │   │
│ │                                                            │   │
│ │ Selected: AWS service                                     │   │
│ │ Service: [EC2 ▼]                                          │   │
│ │ Use case: ○ EC2 (allows EC2 instances to call services)   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 2: Add permissions                                          │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ☑ AmazonS3ReadOnlyAccess                                  │   │
│ │ ☑ CloudWatchLogsFullAccess                                 │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 3: Name, review, and create                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Role name: [EC2-S3-ReadOnly-Role]                          │   │
│ │ Description: [Allows EC2 instances to read from S3]        │   │
│ │                                                            │   │
│ │ Trust policy (auto-generated):                            │   │
│ │ {                                                          │   │
│ │   "Version": "2012-10-17",                                │   │
│ │   "Statement": [{                                          │   │
│ │     "Effect": "Allow",                                     │   │
│ │     "Principal": {                                         │   │
│ │       "Service": "ec2.amazonaws.com"                       │   │
│ │     },                                                     │   │
│ │     "Action": "sts:AssumeRole"                            │   │
│ │   }]                                                       │   │
│ │ }                                                          │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create role]                                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Then attach to EC2:
Portal → EC2 → Instance → Actions → Security → Modify IAM role
Select: EC2-S3-ReadOnly-Role

CLI:
  # Create role
  aws iam create-role \
    --role-name EC2-S3-ReadOnly-Role \
    --assume-role-policy-document file://trust-policy.json

  # Attach permission policy
  aws iam attach-role-policy \
    --role-name EC2-S3-ReadOnly-Role \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

  # Create instance profile (required for EC2)
  aws iam create-instance-profile --instance-profile-name EC2-S3-Profile
  aws iam add-role-to-instance-profile \
    --instance-profile-name EC2-S3-Profile \
    --role-name EC2-S3-ReadOnly-Role

  # Attach to running EC2 instance
  aws ec2 associate-iam-instance-profile \
    --instance-id i-0123456789abcdef0 \
    --iam-instance-profile Name=EC2-S3-Profile
```

### Cross-Account Role

```
Scenario: Account B (Dev) needs to access S3 in Account A (Prod)

Account A (Prod - 111111111111):
  Create role: CrossAccountS3Access
  Trust policy:
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-123"
        }
      }
    }]
  }
  
  Permission policy: S3 read access to specific bucket

Account B (Dev - 222222222222):
  User/Role needs permission to assume the role:
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::111111111111:role/CrossAccountS3Access"
    }]
  }

  Then assume the role:
  aws sts assume-role \
    --role-arn arn:aws:iam::111111111111:role/CrossAccountS3Access \
    --role-session-name dev-user-session \
    --external-id unique-external-id-123

  Returns temporary credentials → use for S3 access in Account A
```

---

## Part 6: MFA (Multi-Factor Authentication)

### Setting Up MFA

```
Portal → IAM → Users → john.doe → Security credentials → MFA

┌─────────────────────────────────────────────────────────────────┐
│                   ASSIGN MFA DEVICE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Device name: [john-phone]                                        │
│                                                                   │
│ MFA device type:                                                 │
│ ├── ○ Authenticator app (TOTP - Google Auth, Authy, etc.)       │
│ │     → Scan QR code → Enter 2 consecutive codes               │
│ │                                                                │
│ ├── ○ Security key (FIDO2 - YubiKey, etc.)                      │
│ │     → Plug in key → Touch to register                        │
│ │                                                                │
│ └── ○ Hardware TOTP token                                       │
│       → Enter serial number + 2 consecutive codes              │
│                                                                   │
│ Recommended: Authenticator app (most convenient)                │
│ Most secure: FIDO2 security key (phishing-resistant)            │
│                                                                   │
│ ⚠️ For ROOT account: ALWAYS enable MFA                           │
│    → Use hardware key if possible                               │
│    → Store backup MFA in secure location                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Enable virtual MFA
  aws iam create-virtual-mfa-device \
    --virtual-mfa-device-name john-phone \
    --outfile QRCode.png \
    --bootstrap-method QRCodePNG

  aws iam enable-mfa-device \
    --user-name john.doe \
    --serial-number arn:aws:iam::123456789012:mfa/john-phone \
    --authentication-code1 123456 \
    --authentication-code2 789012
```

---

## Part 7: Access Keys (Programmatic Access)

### Access Keys Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                   ACCESS KEYS                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Access Key ID + Secret Access Key (credential pair)        │
│ Used by: CLI, SDKs, API calls                                   │
│ Max: 2 per user (to allow rotation)                             │
│                                                                   │
│ ⚠️ SECURITY RULES:                                               │
│ ├── NEVER embed in code                                         │
│ ├── NEVER commit to Git                                         │
│ ├── NEVER share access keys                                     │
│ ├── Rotate every 90 days (enforce via IAM policy)               │
│ ├── Use IAM Roles instead when possible                         │
│ └── Monitor with IAM Access Analyzer                            │
│                                                                   │
│ Better alternatives to access keys:                              │
│ ├── EC2: Use IAM Roles (instance profiles)                     │
│ ├── Lambda: Execution role (automatic)                          │
│ ├── ECS/EKS: Task roles / IRSA                                  │
│ ├── CI/CD: OIDC federation (GitHub Actions, GitLab)             │
│ ├── Local dev: Use AWS SSO (Identity Center) temp creds         │
│ └── External: Cross-account roles                               │
│                                                                   │
│ If you MUST use access keys:                                     │
│ ├── Use IAM credential report for auditing                     │
│ ├── Set up key rotation automation                              │
│ ├── Use aws-vault or similar for secure local storage           │
│ └── Disable/delete unused keys immediately                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Rotation process:
  1. Create new key:      aws iam create-access-key --user-name john.doe
  2. Update applications with new key
  3. Test new key works
  4. Disable old key:     aws iam update-access-key --user-name john.doe \
                            --access-key-id AKIAOLD... --status Inactive
  5. Wait (ensure nothing breaks)
  6. Delete old key:      aws iam delete-access-key --user-name john.doe \
                            --access-key-id AKIAOLD...
```

---

## Part 8: Permission Boundaries

### What are Permission Boundaries?

A Permission Boundary is an advanced feature that sets the **maximum permissions** an IAM entity can have. Even if identity policies grant broad access, the boundary limits the effective permissions.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   PERMISSION BOUNDARIES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Effective Permissions = Identity Policy ∩ Permission Boundary       │
│  (Intersection — must be allowed in BOTH)                            │
│                                                                       │
│  ┌─────────────────────────────────────┐                            │
│  │    Permission Boundary              │                            │
│  │    (max allowed: EC2, S3, Lambda)   │                            │
│  │    ┌─────────────────────┐          │                            │
│  │    │ Identity Policy     │          │                            │
│  │    │ (granted: EC2, RDS) │          │                            │
│  │    │                     │          │                            │
│  │    │  EC2 ← EFFECTIVE    │          │                            │
│  │    │  RDS ← DENIED       │          │                            │
│  │    │  (not in boundary)  │          │                            │
│  │    └─────────────────────┘          │                            │
│  └─────────────────────────────────────┘                            │
│                                                                       │
│  Use case: Delegation                                                │
│  "Let developers create roles for their Lambda functions,            │
│   but ensure those roles can NEVER access IAM, Organizations,        │
│   or billing — even if they write a broad policy."                   │
│                                                                       │
│  Example boundary policy:                                            │
│  {                                                                   │
│    "Version": "2012-10-17",                                          │
│    "Statement": [{                                                   │
│      "Effect": "Allow",                                              │
│      "Action": [                                                     │
│        "s3:*", "dynamodb:*", "lambda:*",                            │
│        "logs:*", "sqs:*", "sns:*"                                   │
│      ],                                                              │
│      "Resource": "*"                                                 │
│    }]                                                                │
│  }                                                                   │
│  → This boundary means: no matter what policies are attached,       │
│    the entity can ONLY do S3, DynamoDB, Lambda, Logs, SQS, SNS     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Attach boundary:
  aws iam put-user-permissions-boundary \
    --user-name john.doe \
    --permissions-boundary arn:aws:iam::123456789012:policy/DeveloperBoundary
```

---

## Part 9: Policy Evaluation Logic

### How AWS Decides Allow or Deny

```
┌─────────────────────────────────────────────────────────────────────┐
│              POLICY EVALUATION LOGIC                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Step 1: Gather all applicable policies                              │
│  ├── Organization SCPs (from all levels)                            │
│  ├── Permission boundaries (if set)                                 │
│  ├── Identity-based policies (user + group + role)                  │
│  ├── Resource-based policies (on the target resource)               │
│  └── Session policies (if using assumed role)                       │
│                                                                       │
│  Step 2: Evaluate                                                    │
│                                                                       │
│           ┌──────────────────┐                                      │
│           │ Is there an      │                                      │
│           │ explicit DENY?   │──── YES ──→ ❌ DENIED                │
│           └────────┬─────────┘                                      │
│                    │ NO                                               │
│                    ▼                                                  │
│           ┌──────────────────┐                                      │
│           │ Is there an      │                                      │
│           │ SCP that allows? │──── NO ───→ ❌ DENIED                │
│           │ (if in Org)      │                                      │
│           └────────┬─────────┘                                      │
│                    │ YES                                              │
│                    ▼                                                  │
│           ┌──────────────────┐                                      │
│           │ Is it within     │                                      │
│           │ Permission       │──── NO ───→ ❌ DENIED                │
│           │ Boundary?        │                                      │
│           └────────┬─────────┘                                      │
│                    │ YES (or no boundary set)                         │
│                    ▼                                                  │
│           ┌──────────────────┐                                      │
│           │ Is there an      │                                      │
│           │ explicit ALLOW?  │──── NO ───→ ❌ DENIED (implicit)     │
│           └────────┬─────────┘                                      │
│                    │ YES                                              │
│                    ▼                                                  │
│               ✅ ALLOWED                                             │
│                                                                       │
│  KEY RULES:                                                          │
│  1. Default: IMPLICIT DENY (everything denied unless allowed)        │
│  2. Explicit DENY always wins (overrides any Allow)                  │
│  3. Must have explicit Allow AND not be denied by boundary/SCP      │
│  4. Resource-based policies can grant cross-account access           │
│     (without identity policy on the other side)                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: AWS Identity Center (SSO) — The Modern Way

### Why Identity Center?

```
┌─────────────────────────────────────────────────────────────────┐
│              IAM USERS vs IDENTITY CENTER                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Old Way (IAM Users):                                            │
│ ├── Create user in each account                                 │
│ ├── Manage passwords per account                                │
│ ├── Long-lived access keys                                      │
│ ├── No centralized user management                              │
│ └── ⚠️ Doesn't scale for organizations                          │
│                                                                   │
│ New Way (Identity Center / SSO):                                │
│ ├── Single sign-on across ALL AWS accounts                      │
│ ├── Centralized user/group management                           │
│ ├── Temporary credentials (auto-rotate)                         │
│ ├── Connect to corporate directory (Active Directory, Okta)     │
│ ├── Permission Sets (reusable role templates)                   │
│ └── ✅ Recommended for ALL organizations                        │
│                                                                   │
│ Flow:                                                            │
│ User → SSO Portal → Select Account → Get temp creds → Work     │
│                                                                   │
│ CLI flow:                                                        │
│ aws configure sso → Login via browser → Get temp session        │
│ aws sso login --profile prod-admin                              │
│ aws s3 ls --profile prod-admin (uses temp creds automatically)  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Setting Up Identity Center

```
Portal → AWS Identity Center → Enable

┌─────────────────────────────────────────────────────────────────┐
│                IDENTITY CENTER SETUP                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Choose identity source                                   │
│ ├── Identity Center directory (built-in — good for small orgs) │
│ ├── Active Directory (AD Connector or AWS Managed AD)           │
│ └── External IdP (Okta, Azure AD, Google, OneLogin)             │
│                                                                   │
│ Step 2: Create Permission Sets (reusable role templates)        │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Permission Set: AdministratorAccess                        │   │
│ │ Policy: AWS managed - AdministratorAccess                 │   │
│ │ Session duration: 4 hours                                  │   │
│ │                                                            │   │
│ │ Permission Set: DeveloperAccess                            │   │
│ │ Policy: Custom - EC2, S3, Lambda, RDS, DynamoDB           │   │
│ │ Session duration: 8 hours                                  │   │
│ │                                                            │   │
│ │ Permission Set: ReadOnlyAccess                             │   │
│ │ Policy: AWS managed - ViewOnlyAccess                      │   │
│ │ Session duration: 12 hours                                 │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 3: Assign users/groups → accounts → permission sets        │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Group: DevOps-Team                                         │   │
│ │ → Account: Production (111111111111)                      │   │
│ │   → Permission Set: AdministratorAccess                   │   │
│ │ → Account: Staging (222222222222)                         │   │
│ │   → Permission Set: AdministratorAccess                   │   │
│ │ → Account: Development (333333333333)                     │   │
│ │   → Permission Set: AdministratorAccess                   │   │
│ │                                                            │   │
│ │ Group: Developers                                          │   │
│ │ → Account: Development (333333333333)                     │   │
│ │   → Permission Set: DeveloperAccess                       │   │
│ │ → Account: Production (111111111111)                      │   │
│ │   → Permission Set: ReadOnlyAccess                        │   │
│ │                                                            │   │
│ │ Group: Finance                                             │   │
│ │ → Account: Management (444444444444)                      │   │
│ │   → Permission Set: BillingAccess                         │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ SSO Portal URL: https://d-xxxxxxxx.awsapps.com/start            │
│ Users log in here to access all assigned accounts               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 11: IAM Security Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                IAM SECURITY BEST PRACTICES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. ROOT ACCOUNT                                                     │
│  ├── Enable MFA (hardware key preferred)                            │
│  ├── Never create access keys                                       │
│  ├── Never use for daily tasks                                      │
│  └── Store credentials in physical safe                             │
│                                                                       │
│  2. PRINCIPLE OF LEAST PRIVILEGE                                     │
│  ├── Start with zero permissions, add as needed                     │
│  ├── Use IAM Access Analyzer to identify unused permissions         │
│  ├── Use IAM policy simulator to test before applying              │
│  ├── Regularly review and remove unnecessary permissions            │
│  └── Use condition keys to restrict (region, time, source IP)       │
│                                                                       │
│  3. USE ROLES OVER USERS                                             │
│  ├── EC2/Lambda/ECS → use IAM roles (no long-lived keys)           │
│  ├── Cross-account → use roles (not shared users)                   │
│  ├── Federation → use roles (for SSO)                               │
│  └── CI/CD → use OIDC federation (not access keys)                  │
│                                                                       │
│  4. ENFORCE MFA                                                      │
│  ├── All human users must have MFA                                  │
│  ├── Require MFA for sensitive operations (via policy condition)    │
│  └── Consider FIDO2 keys for high-privilege users                   │
│                                                                       │
│  5. CREDENTIAL MANAGEMENT                                            │
│  ├── Rotate access keys every 90 days                               │
│  ├── Use AWS Secrets Manager for application secrets                │
│  ├── Never hardcode credentials in code                             │
│  ├── Use credential report for auditing                             │
│  └── Enable IAM Access Analyzer for external access findings       │
│                                                                       │
│  6. MONITORING                                                       │
│  ├── Enable CloudTrail in all regions                               │
│  ├── Set up alerts for: root login, IAM changes, key creation       │
│  ├── Review IAM credential report monthly                           │
│  └── Use IAM Access Advisor to see last-used permissions           │
│                                                                       │
│  7. ORGANIZATION                                                     │
│  ├── Use IAM Groups (never attach policies directly to users)       │
│  ├── Use Identity Center (SSO) for multi-account                    │
│  ├── Use SCPs for guardrails across accounts                        │
│  └── Use permission boundaries for delegated administration         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Real-World Patterns

### Startup (5-10 Developers)

```
Setup:
├── Enable Identity Center (even with 1 account, plan ahead)
├── Create groups: Admins, Developers
├── Permission Sets:
│   ├── Admin: AdministratorAccess (CTO + senior dev)
│   └── Developer: PowerUserAccess minus IAM (all devs)
├── Require MFA for all users
├── EC2/Lambda: Always use IAM roles
└── No access keys for humans (use SSO CLI)

IAM roles for services:
├── EC2-WebApp-Role: S3 read + CloudWatch logs + Secrets Manager
├── Lambda-ProcessOrder-Role: DynamoDB + SQS + CloudWatch
└── CI-CD-Role (GitHub OIDC): ECR push + ECS deploy + S3
```

### Mid-Size Company (50-100 Developers)

```
Setup:
├── AWS Organizations with multiple accounts
│   ├── Management, Production, Staging, Dev, Shared
├── Identity Center connected to corporate IdP (Okta/Azure AD)
├── Permission Sets per role:
│   ├── PlatformAdmin: Full access (platform team)
│   ├── DeveloperProd: Read-only in prod
│   ├── DeveloperDev: Full in dev
│   ├── DBA: RDS/DynamoDB in data accounts
│   └── SecurityAudit: Security audit read
├── SCPs:
│   ├── Deny regions outside ap-south-1, us-east-1
│   ├── Deny leaving organization
│   ├── Require encryption on S3/EBS
│   └── Deny root account usage
├── Permission boundaries for team-created roles
└── Monthly access review (IAM Access Analyzer)
```

### Enterprise (500+ Developers)

```
Setup:
├── AWS Organizations with 50+ accounts
├── Identity Center federated to Active Directory
├── Landing Zone with Control Tower
├── Layered access model:
│   ├── L1: Platform Team → All accounts (admin)
│   ├── L2: Team Leads → Their team's accounts (contributor)
│   ├── L3: Developers → Dev/staging (write), Prod (read-only)
│   ├── L4: Auditors → All accounts (read + security tools)
│   └── L5: Business → Billing/cost only
├── Automated access provisioning:
│   ├── HR system → IdP → Identity Center (auto-sync)
│   ├── Team change → Auto-update group membership
│   └── Offboarding → Automatic revocation within 1 hour
├── Advanced controls:
│   ├── Permission boundaries on all dev-created roles
│   ├── Session policies for contractors
│   ├── IP-based restrictions for sensitive accounts
│   ├── Time-based access (break-glass for prod changes)
│   └── IAM Access Analyzer running continuously
└── Compliance reporting:
    ├── Monthly credential reports
    ├── Quarterly access reviews
    └── Annual penetration testing of IAM config
```

---

## Troubleshooting: Common IAM Issues

### "AccessDenied" — The Most Common AWS Error

```
Debugging steps:
1. ☐ Which identity is making the call?
   Run: aws sts get-caller-identity

2. ☐ Does the identity have the required permission?
   Use IAM Policy Simulator: Console → IAM → Policy Simulator
   Select the user/role → Select the service + action → Run Simulation

3. ☐ Is there an explicit DENY anywhere?
   Check: identity policies, resource policies, SCPs, permission boundaries
   (Explicit DENY always wins over any Allow)

4. ☐ Is an SCP blocking it?
   Console → Organizations → Policies → check account's effective SCPs

5. ☐ Is the resource in the right region/account?
   Cross-account access requires a role trust policy
```

### Worked Example: Policy Evaluation

**Scenario:** User `john.doe` is in the `Developers` group with `AmazonEC2FullAccess`. There's an SCP denying `eu-west-1`. John tries to launch an EC2 in `eu-west-1`.

```
Step 1: Is there an explicit deny?  → YES (SCP denies eu-west-1)
Step 2: SCP deny overrides everything
Result: ❌ AccessDenied (even though IAM policy allows EC2FullAccess)

Lesson: SCPs are guardrails that ALWAYS win.
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using root account for daily work | No audit trail, maximum blast radius | Create IAM users, lock root with MFA |
| Hardcoding access keys in code | Keys leak via Git, get compromised | Use IAM roles (EC2 instance profiles, Lambda execution roles) |
| Not rotating access keys | Old keys are security risks | Rotate every 90 days, prefer roles |
| Overly broad policies (`*:*`) | Any compromised identity can do anything | Follow least privilege, use specific actions |
| Forgetting to attach policies | User/role has no permissions | Check: IAM → User → Permissions tab |

---

## Quick Reference: CLI Commands

```bash
# Users
aws iam create-user --user-name NAME
aws iam delete-user --user-name NAME
aws iam list-users

# Groups
aws iam create-group --group-name NAME
aws iam add-user-to-group --group-name GROUP --user-name USER
aws iam attach-group-policy --group-name GROUP --policy-arn ARN

# Policies
aws iam create-policy --policy-name NAME --policy-document file://policy.json
aws iam attach-user-policy --user-name USER --policy-arn ARN
aws iam list-attached-user-policies --user-name USER

# Roles
aws iam create-role --role-name NAME --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name NAME --policy-arn ARN
aws sts assume-role --role-arn ARN --role-session-name SESSION

# Access Keys
aws iam create-access-key --user-name USER
aws iam list-access-keys --user-name USER
aws iam update-access-key --user-name USER --access-key-id KEY --status Inactive
aws iam delete-access-key --user-name USER --access-key-id KEY

# Security
aws iam get-credential-report
aws iam generate-credential-report
aws iam get-account-summary
```

## Terraform Examples

```hcl
# IAM User
resource "aws_iam_user" "developer" {
  name = "john.doe"
  tags = { Team = "backend" }
}

# IAM Group
resource "aws_iam_group" "developers" {
  name = "Developers"
}

# Add user to group
resource "aws_iam_group_membership" "dev_members" {
  name  = "dev-membership"
  users = [aws_iam_user.developer.name]
  group = aws_iam_group.developers.name
}

# Attach managed policy to group
resource "aws_iam_group_policy_attachment" "dev_ec2" {
  group      = aws_iam_group.developers.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2FullAccess"
}

# Custom policy
resource "aws_iam_policy" "s3_read_only" {
  name = "S3ReadOnlyCustom"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:ListBucket"]
        Resource = ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
      }
    ]
  })
}

# IAM Role for EC2 (instance profile)
resource "aws_iam_role" "ec2_app" {
  name = "ec2-app-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = { Service = "ec2.amazonaws.com" }
        Action    = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2_app" {
  name = "ec2-app-profile"
  role = aws_iam_role.ec2_app.name
}
```

## IAM Limits Quick Reference

| Entity | Max per Account | Key Fact |
|--------|----------------|----------|
| Users | 5,000 | Use IAM Identity Center for >50 users |
| Groups | 300 | Max 10 groups per user |
| Roles | 1,000 | Soft limit, can request increase |
| Managed Policies (custom) | 1,500 | Max 10 per user/group/role |
| Inline Policies | Unlimited | Avoid — use managed policies |
| Access Keys | 2 per user | Rotate every 90 days, prefer roles |
| MFA devices | 8 per user | Enable on root + all human users |

---

## What's Next?

In the next chapter, we'll cover AWS Billing & Cost Management — pricing models, Cost Explorer, budgets, savings plans, and cost optimization strategies.

→ Next: [Chapter 5: Billing & Cost Management](05-billing.md)

---

*Last Updated: May 2026*
