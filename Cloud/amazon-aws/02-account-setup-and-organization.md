# Chapter 2: Account Setup & Organization (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Creating a Personal AWS Account (Free Tier)](#part-1-creating-a-personal-aws-account-free-tier)
- [Part 2: Understanding the Root User vs IAM Users](#part-2-understanding-the-root-user-vs-iam-users)
- [Part 3: AWS Organizations (Multi-Account Management)](#part-3-aws-organizations-multi-account-management)
- [Part 4: Service Control Policies (SCPs)](#part-4-service-control-policies-scps)
- [Part 5: Consolidated Billing](#part-5-consolidated-billing)
- [Part 6: AWS IAM Identity Center (SSO)](#part-6-aws-iam-identity-center-sso)

---

## Overview

### Why Does Account Setup Matter?

> **Real-World Analogy:** Think of an AWS Organization like a corporate office building. The Management Account is the building owner who pays rent and sets building rules. Each AWS Account is a separate office suite with its own locks and keys. Departments (OUs) are floors — Engineering on floor 3, Finance on floor 5. Service Control Policies are building rules everyone must follow, like "no smoking" — even if your office allows it.

Even if you're just learning, understanding multi-account structure is critical because **every real company uses it**. You'll encounter this in your first week at any cloud-native job. And for personal use, a misconfigured root account can lead to a surprise $10,000 bill from a compromised credential.

This chapter covers everything about creating and managing AWS accounts — from a personal free-tier account for learning, to a full enterprise multi-account organization setup used by companies in production.

```
What you'll learn:
├── Creating your personal AWS account (free tier)
├── AWS Organizations (multi-account management)
├── Organizational Units (OUs) structure
├── Service Control Policies (SCPs)
├── Consolidated billing
├── AWS IAM Identity Center (SSO)
├── Account security best practices (Day 1)
├── How companies actually set this up
└── Step-by-step portal walkthrough
```

---

## Part 1: Creating a Personal AWS Account (Free Tier)

### What You Need Before Starting

| Requirement | Details |
|-------------|---------|
| Email address | Unique email (not used for another AWS account). Use `+` trick: `yourname+aws@gmail.com` |
| Phone number | For verification (SMS or call) |
| Credit/Debit card | Required even for free tier (charged $1 temporarily for verification) |
| Account name | Any name (e.g., "John Personal AWS") |

### Step-by-Step: Creating an Account

```
Portal: https://aws.amazon.com → "Create an AWS Account"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: Root User Email & Account Name
┌─────────────────────────────────────────────────────┐
│ Root user email address: [your-email@domain.com]    │
│ AWS account name:        [My Learning Account]      │
│                                                     │
│ → "Verify email address" (OTP sent to email)        │
└─────────────────────────────────────────────────────┘

Step 2: Root User Password
┌─────────────────────────────────────────────────────┐
│ Password: [Strong password - min 8 chars]           │
│ Confirm:  [Same password]                           │
│                                                     │
│ ⚠️ SAVE THIS! Root user = God-level access.         │
│    Store in password manager immediately.           │
└─────────────────────────────────────────────────────┘

Step 3: Contact Information
┌─────────────────────────────────────────────────────┐
│ Account type: ○ Personal  ○ Business                │
│                                                     │
│ Full name:    [Your Name]                           │
│ Phone:        [+91-XXXXXXXXXX]                      │
│ Country:      [India]                               │
│ Address:      [Your Address]                        │
│ City:         [City]                                │
│ State:        [State]                               │
│ Postal Code:  [XXXXXX]                              │
│                                                     │
│ 💡 For learning: "Personal" is fine                  │
│ 💡 For company: "Business" (needed for support)     │
└─────────────────────────────────────────────────────┘

Step 4: Payment Information
┌─────────────────────────────────────────────────────┐
│ Credit/Debit Card Number: [XXXX-XXXX-XXXX-XXXX]    │
│ Expiration:               [MM/YY]                   │
│ Cardholder Name:          [Name on Card]            │
│                                                     │
│ ⚠️ $1 (or ₹2) temporary hold for verification      │
│    Refunded within 3-5 days                         │
│                                                     │
│ 💡 You can set a billing alarm to avoid surprises   │
└─────────────────────────────────────────────────────┘

Step 5: Identity Verification
┌─────────────────────────────────────────────────────┐
│ Verification method: ○ Text message  ○ Voice call   │
│ Phone number: [+91-XXXXXXXXXX]                      │
│ Security check: [CAPTCHA]                           │
│                                                     │
│ → Enter verification code received                  │
└─────────────────────────────────────────────────────┘

Step 6: Select a Support Plan
┌─────────────────────────────────────────────────────┐
│ ○ Basic Support (Free)          ← SELECT THIS       │
│ ○ Developer ($29/month)                             │
│ ○ Business ($100/month min)                         │
│ ○ Enterprise On-Ramp ($5,500/month)                 │
│ ○ Enterprise ($15,000/month)                        │
│                                                     │
│ 💡 Basic is fine for learning. Upgrade later        │
│    if needed for production.                        │
└─────────────────────────────────────────────────────┘

✅ Account Created! Takes 1-2 minutes to activate fully.
```

### Immediately After Account Creation (Security Checklist)

```
⚠️  DAY 1 SECURITY TASKS (Do these IMMEDIATELY):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

□ 1. Enable MFA on Root User
     Console → IAM → Security credentials → MFA → Assign MFA device
     Use: Authenticator app (Google Authenticator, Authy) or hardware key
     
□ 2. Create an IAM User for daily use
     Console → IAM → Users → Create user
     - Username: "admin" or your name
     - Attach policy: AdministratorAccess
     - Enable Console access
     - Enable MFA on this user too
     
□ 3. Set up a Billing Alarm
     Console → Billing → Budgets → Create budget
     - Budget type: Cost budget
     - Amount: $5 (or whatever your limit is)
     - Alert at: 80% and 100%
     - Email notification: your email
     
□ 4. Never use Root user again for daily tasks
     Root user should ONLY be used for:
     - Changing account settings
     - Closing the account
     - Changing support plan
     - Restoring IAM permissions (if locked out)

□ 5. Enable CloudTrail (if not already on)
     Console → CloudTrail → Create trail
     - Trail name: "management-events"
     - Apply to all regions: Yes
     - Log: Management events
     
□ 6. Configure Account-Level Settings
     Console → Account → Alternate contacts
     - Add billing, operations, security contacts
     Console → Account → IAM User and Role Access to Billing
     - Enable (so IAM users can see billing)
```

---

## Part 2: Understanding the Root User vs IAM Users

```
┌─────────────────────────────────────────────────────────────────┐
│                     ROOT USER vs IAM USER                         │
├──────────────────────────────┬──────────────────────────────────┤
│        ROOT USER             │         IAM USER                  │
├──────────────────────────────┼──────────────────────────────────┤
│ Created with the account     │ Created by root or other IAM     │
│ Email + Password login       │ Account ID + Username + Password │
│ CANNOT be restricted         │ CAN be restricted via policies   │
│ Full unrestricted access     │ Only has assigned permissions    │
│ Can close the account        │ Cannot close the account         │
│ Can change support plan      │ Cannot change support plan       │
│ Can enable/disable regions   │ Cannot manage regions            │
│                              │                                  │
│ ⚠️ NEVER use for daily work  │ ✅ Use this for everything else  │
│ 🔒 Lock it away with MFA     │ 🔑 Individual user per person    │
└──────────────────────────────┴──────────────────────────────────┘

How to sign in:

Root User:
  URL: https://console.aws.amazon.com
  Login: Email + Password + MFA

IAM User:
  URL: https://<account-id>.signin.aws.amazon.com/console
  Login: Account ID + Username + Password + MFA
  (or use custom sign-in URL: https://<alias>.signin.aws.amazon.com/console)
```

---

## Part 3: AWS Organizations (Multi-Account Management)

### What is AWS Organizations?

AWS Organizations lets you centrally manage multiple AWS accounts from a single **management account** (formerly called "master account").

```
Why use multiple accounts?

Single Account Problems:
├── Blast radius — one mistake affects everything
├── Hard to track costs per team/project
├── Service limits shared across all workloads
├── Complex IAM — who can access what?
├── Compliance nightmare — mixing dev and prod
└── Security — one breach compromises all

Multi-Account Benefits:
├── Isolation — dev can't accidentally delete prod
├── Clear billing — cost per account/team/project
├── Independent service limits per account
├── Simple IAM — each account has minimal policies
├── Compliance — sensitive data in separate account
└── Security — blast radius is limited to one account
```

### Creating an AWS Organization

```
Portal: Console → AWS Organizations → Create organization
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: Create Organization
┌─────────────────────────────────────────────────────────┐
│ The current account becomes the MANAGEMENT ACCOUNT      │
│ (This cannot be changed later!)                         │
│                                                         │
│ Feature set:                                            │
│ ○ All features (recommended) ← SELECT THIS             │
│   - Includes consolidated billing                      │
│   - Plus SCPs, tag policies, AI opt-out policies       │
│ ○ Consolidated billing only                            │
│   - Just billing aggregation, no governance            │
│                                                         │
│ [Create organization]                                   │
└─────────────────────────────────────────────────────────┘

Step 2: Verify management account email (if not done)

Step 3: Organization is created!
- Organization ID: o-xxxxxxxxxx
- Root ID: r-xxxx
- Management Account ID: 123456789012
```

### Organization Structure Diagram

```
AWS Organization
│
├── Root (r-xxxx)
│   │
│   │── Management Account (123456789012) ← NEVER deploy workloads here
│   │   └── Used for: Billing, Organization management, SCPs
│   │
│   ├── OU: Security ──────────────── SCP: DenyAllExcept Security tools
│   │   ├── Account: Log Archive (111111111111)
│   │   │   └── S3 buckets for CloudTrail, Config, VPC Flow Logs
│   │   └── Account: Security Tooling (222222222222)
│   │       └── GuardDuty admin, Security Hub, Inspector
│   │
│   ├── OU: Infrastructure ─────────── SCP: Deny region restrictions
│   │   ├── Account: Networking (333333333333)
│   │   │   └── Transit Gateway, Direct Connect, Route 53
│   │   └── Account: Shared Services (444444444444)
│   │       └── CI/CD, Container Registry, Artifact Store
│   │
│   ├── OU: Workloads
│   │   ├── OU: Production ─────────── SCP: Deny delete without MFA
│   │   │   ├── Account: Prod-Frontend (555555555555)
│   │   │   ├── Account: Prod-Backend (666666666666)
│   │   │   └── Account: Prod-Data (777777777777)
│   │   │
│   │   ├── OU: Staging ────────────── SCP: Allowed regions only
│   │   │   └── Account: Staging (888888888888)
│   │   │
│   │   └── OU: Development ────────── SCP: Budget limits, no production services
│   │       └── Account: Development (999999999999)
│   │
│   └── OU: Sandbox ───────────────── SCP: Max spend limit, limited services
│       ├── Account: Sandbox-Dev1
│       └── Account: Sandbox-Dev2
│
└── Service Control Policies (SCPs) applied at OU/Account level
    └── Inherited downward (OU → Child OU → Account)
```

### Adding Accounts to Organization

```
Two ways to add accounts:

Method 1: CREATE a new account from Organization
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Console → Organizations → Add an AWS account → Create an AWS account

┌─────────────────────────────────────────────────────────┐
│ AWS account name:     [Prod-Backend]                    │
│ Email of account owner: [prod-backend@company.com]      │
│                                                         │
│ ⚠️ Each account needs a UNIQUE email address             │
│ 💡 Use email aliases: aws+prod-backend@company.com      │
│                                                         │
│ IAM role name: OrganizationAccountAccessRole (default)  │
│ └── This role lets management account assume into it    │
└─────────────────────────────────────────────────────────┘

Method 2: INVITE an existing account
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Console → Organizations → Add an AWS account → Invite an existing account

┌─────────────────────────────────────────────────────────┐
│ Email or Account ID: [target-account@company.com]       │
│ or Account ID:       [123456789012]                     │
│ Notes (optional):    [Join our org please]              │
│                                                         │
│ → Invitation sent. Target account must accept.          │
│ ⚠️ Invited account's root user must accept invitation   │
└─────────────────────────────────────────────────────────┘
```

### Creating Organizational Units (OUs)

```
Console → Organizations → Root → Actions → Create organizational unit

┌─────────────────────────────────────────────────────────┐
│ Organizational unit name: [Security]                    │
│ Parent: Root                                            │
│                                                         │
│ [Create organizational unit]                            │
└─────────────────────────────────────────────────────────┘

Then move accounts into OUs:
Console → Organizations → Select account → Actions → Move
→ Select destination OU
```

---

## Part 4: Service Control Policies (SCPs)

### What are SCPs?

SCPs are guardrails that define the **maximum permissions** for accounts in an organization. They don't grant permissions — they restrict what IAM policies CAN grant.

```
How SCPs Work:

    IAM Policy says: "Allow EC2:*"
    SCP says: "Deny EC2 in eu-west-1"
    
    Result: User can use EC2 everywhere EXCEPT eu-west-1
    
    Think of it as:
    ┌──────────────────────────────────────────────┐
    │  Effective Permission = IAM Policy ∩ SCP     │
    │  (Intersection — must be allowed by BOTH)    │
    └──────────────────────────────────────────────┘

    SCP does NOT affect management account!
    SCP affects ALL users/roles in member accounts (including root user of that account)
```

### SCP Diagram

```
Permission Flow:

  ┌─────────┐     ┌─────────┐     ┌───────────────┐
  │   SCP   │     │   IAM   │     │   Effective   │
  │ (Fence) │  ∩  │ (Grant) │  =  │  Permission   │
  └─────────┘     └─────────┘     └───────────────┘

Example: Region Restriction SCP

  SCP allows: us-east-1, us-west-2, ap-south-1 only
  
  ┌──────────────────────────────┐
  │     ALL AWS REGIONS          │
  │  ┌────────────────────────┐  │
  │  │  SCP ALLOWED REGIONS   │  │
  │  │  ┌──────────────────┐  │  │
  │  │  │ IAM permissions  │  │  │  ← Only THIS area is effective
  │  │  └──────────────────┘  │  │
  │  └────────────────────────┘  │
  │                              │
  │  Blocked by SCP (even if    │
  │  IAM says Allow)            │
  └──────────────────────────────┘
```

### Common SCP Examples

```
Console → Organizations → Policies → Service control policies → Create policy

1. DENY ACCESS TO UNUSED REGIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "ap-south-1",
            "eu-west-1"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/OrganizationAccountAccessRole"
        }
      }
    }
  ]
}
```

```
2. PREVENT LEAVING THE ORGANIZATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeaveOrg",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

```
3. REQUIRE ENCRYPTION ON S3
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedS3",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

```
4. DENY DISABLING CLOUDTRAIL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCloudTrailDisable",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

### Attaching SCPs

```
Console → Organizations → Policies → Service control policies
→ Select your policy → Targets tab → Attach

┌─────────────────────────────────────────────────────────┐
│ Attach to:                                              │
│ ○ Root (applies to ALL accounts & OUs)                 │
│ ○ Specific OU (applies to all accounts in that OU)     │
│ ○ Specific Account                                     │
│                                                         │
│ 💡 Best practice: Attach to OUs, not individual accounts│
│                                                         │
│ ⚠️ Default SCP: "FullAWSAccess" is attached to Root     │
│    If you remove it → ALL permissions denied!           │
│    Use DENY policies instead of removing this.          │
└─────────────────────────────────────────────────────────┘
```

---

## Part 5: Consolidated Billing

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                   CONSOLIDATED BILLING                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Management Account (Payer Account)                          │
│  └── Receives ONE bill for all member accounts               │
│                                                               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Acct: A │  │ Acct: B │  │ Acct: C │  │ Acct: D │      │
│  │ $200    │  │ $150    │  │ $400    │  │ $100    │      │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │
│       │              │            │            │              │
│       └──────────────┴────────────┴────────────┘              │
│                           │                                   │
│              ┌────────────▼────────────┐                     │
│              │  Combined Bill: $850    │                     │
│              │  (Single payment method)│                     │
│              └─────────────────────────┘                     │
│                                                               │
│  Benefits:                                                   │
│  ├── Volume discounts (combined usage across accounts)       │
│  ├── S3: Combined storage = higher tier pricing              │
│  ├── EC2: Reserved Instances shared across accounts          │
│  ├── Savings Plans: Shared across accounts                   │
│  ├── Single payment method for all accounts                  │
│  └── Cost allocation tags for detailed breakdown             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Cost Allocation Setup

```
Console (Management Account) → Billing → Cost allocation tags

┌─────────────────────────────────────────────────────────┐
│ AWS-generated tags:    [✓ Activate]                     │
│ - aws:createdBy                                         │
│                                                         │
│ User-defined tags:     [✓ Activate]                     │
│ - Environment (prod/staging/dev)                        │
│ - Team (frontend/backend/data)                          │
│ - Project (project-name)                                │
│ - CostCenter (department code)                          │
│                                                         │
│ 💡 Takes 24 hours to appear in billing reports          │
│ 💡 Tags MUST be applied to resources for tracking       │
└─────────────────────────────────────────────────────────┘
```

---

## Part 6: AWS IAM Identity Center (SSO)

### What is It?

IAM Identity Center (formerly AWS SSO) provides **single sign-on** access to all your AWS accounts and business applications.

```
Without SSO:                          With SSO:
━━━━━━━━━━━━━                         ━━━━━━━━━
Login to Account A (username/pass)    Login ONCE to SSO Portal
Login to Account B (username/pass)    → See all accounts
Login to Account C (username/pass)    → Click to access any account
Login to Account D (username/pass)    → Automatic role assumption

Nightmare for 20+ accounts!           One login, access everything!
```

### Setting Up IAM Identity Center

```
Console → IAM Identity Center → Enable

Step 1: Choose Identity Source
┌─────────────────────────────────────────────────────────┐
│ Identity source:                                        │
│ ○ Identity Center directory (built-in) ← Start here    │
│ ○ Active Directory (AD Connector / AWS Managed AD)      │
│ ○ External IdP (Okta, Azure AD, Google Workspace)       │
│                                                         │
│ 💡 For company: Use External IdP (Okta/Azure AD)        │
│ 💡 For personal/small team: Built-in is fine            │
└─────────────────────────────────────────────────────────┘

Step 2: Create Users (if using built-in)
┌─────────────────────────────────────────────────────────┐
│ Console → IAM Identity Center → Users → Add user       │
│                                                         │
│ Username:      [john.doe]                               │
│ Email:         [john.doe@company.com]                   │
│ First name:    [John]                                   │
│ Last name:     [Doe]                                    │
│ Display name:  [John Doe]                               │
│                                                         │
│ → User receives email to set password                   │
└─────────────────────────────────────────────────────────┘

Step 3: Create Groups
┌─────────────────────────────────────────────────────────┐
│ Console → IAM Identity Center → Groups → Create group   │
│                                                         │
│ Group name: [Developers]                                │
│ Add members: [John Doe, Jane Smith, ...]                │
│                                                         │
│ Typical groups:                                         │
│ ├── Admins (full access to all accounts)               │
│ ├── Developers (access to dev/staging accounts)         │
│ ├── DevOps (access to all accounts, infra permissions) │
│ ├── ReadOnly (view access to prod for monitoring)      │
│ └── DataEngineers (access to data accounts)            │
└─────────────────────────────────────────────────────────┘

Step 4: Create Permission Sets
┌─────────────────────────────────────────────────────────┐
│ Console → IAM Identity Center → Permission sets        │
│ → Create permission set                                │
│                                                         │
│ Type:                                                   │
│ ○ Predefined (AWS managed policies)                    │
│   └── AdministratorAccess, PowerUserAccess,            │
│       ViewOnlyAccess, ReadOnlyAccess, etc.             │
│ ○ Custom                                               │
│   └── Define specific policy (e.g., EC2+S3 only)       │
│                                                         │
│ Session duration: [1 hour / 4 hours / 8 hours / 12h]   │
│                                                         │
│ Common Permission Sets:                                 │
│ ├── AdminAccess (AdministratorAccess)                  │
│ ├── PowerUser (everything except IAM/Org management)    │
│ ├── DeveloperAccess (custom: compute+storage+db)       │
│ ├── ReadOnly (ViewOnlyAccess)                          │
│ └── BillingAccess (Billing policy)                     │
└─────────────────────────────────────────────────────────┘

Step 5: Assign Users/Groups to Accounts with Permission Sets
┌─────────────────────────────────────────────────────────┐
│ Console → IAM Identity Center → AWS accounts           │
│ → Select account(s) → Assign users or groups           │
│                                                         │
│ Example Assignments:                                    │
│                                                         │
│ Group: Admins                                           │
│ ├── Account: Management → AdminAccess                  │
│ ├── Account: Prod-All → AdminAccess                    │
│ └── Account: Dev-All → AdminAccess                     │
│                                                         │
│ Group: Developers                                       │
│ ├── Account: Dev → PowerUser                           │
│ ├── Account: Staging → ReadOnly                        │
│ └── Account: Prod → ReadOnly                           │
│                                                         │
│ Group: DevOps                                           │
│ ├── Account: All accounts → PowerUser                  │
│ └── Account: Infrastructure → AdminAccess              │
└─────────────────────────────────────────────────────────┘
```

### SSO Portal Experience

```
After setup, users access: https://<your-domain>.awsapps.com/start

┌─────────────────────────────────────────────────────────┐
│  AWS Access Portal                                      │
│                                                         │
│  Your AWS Accounts:                                     │
│  ┌───────────────────────────────────────────────────┐ │
│  │ 📁 Development (999999999999)                     │ │
│  │    ├── PowerUserAccess    [Management Console]    │ │
│  │    │                      [Command line access]   │ │
│  │    └── Programmatic/CLI credentials shown here    │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ 📁 Staging (888888888888)                         │ │
│  │    └── ReadOnlyAccess     [Management Console]    │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ 📁 Production (555555555555)                      │ │
│  │    └── ReadOnlyAccess     [Management Console]    │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  Click "Management Console" → Opens that account's     │
│  console with the assigned role assumed automatically.  │
│                                                         │
│  Click "Command line access" → Shows temporary         │
│  credentials for CLI/SDK use.                           │
└─────────────────────────────────────────────────────────┘
```

---

## Part 7: AWS Support Plans

```
┌──────────────────────────────────────────────────────────────────────┐
│                        AWS SUPPORT PLANS                               │
├──────────┬──────────┬──────────┬───────────────┬────────────────────┤
│          │ Basic    │Developer │ Business      │ Enterprise         │
├──────────┼──────────┼──────────┼───────────────┼────────────────────┤
│ Cost     │ Free     │ $29/mo   │ $100/mo or 3% │ $15,000/mo or 3%  │
│          │          │          │ of spend      │ of spend           │
├──────────┼──────────┼──────────┼───────────────┼────────────────────┤
│ Tech     │ None     │ Email    │ 24/7 Phone    │ 24/7 Phone         │
│ Support  │          │ (12h)    │ Chat, Email   │ Chat, Email        │
├──────────┼──────────┼──────────┼───────────────┼────────────────────┤
│ Response │ N/A      │ 12-24h   │ 1h (urgent)   │ 15 min (critical) │
├──────────┼──────────┼──────────┼───────────────┼────────────────────┤
│ TAM      │ No       │ No       │ No            │ Yes (dedicated)    │
├──────────┼──────────┼──────────┼───────────────┼────────────────────┤
│ Trusted  │ Core     │ Core     │ Full          │ Full               │
│ Advisor  │ checks   │ checks   │ checks        │ checks             │
├──────────┼──────────┼──────────┼───────────────┼────────────────────┤
│ Best for │ Learning │ Dev/Test │ Production    │ Enterprise/Critical│
└──────────┴──────────┴──────────┴───────────────┴────────────────────┘
```

---

## Part 8: AWS Control Tower (Automated Multi-Account Setup)

### What is Control Tower?

AWS Control Tower automates the setup of a secure, well-architected multi-account environment based on AWS best practices.

```
Without Control Tower:                 With Control Tower:
━━━━━━━━━━━━━━━━━━━━━                  ━━━━━━━━━━━━━━━━━━━━
Manually create org                    One-click landing zone setup
Manually create OUs                    Pre-configured OUs created
Manually create accounts               Account Factory (vending machine)
Manually write SCPs                    Pre-built guardrails (preventive + detective)
Manually set up CloudTrail             Auto-configured audit trail
Manually set up Config                 Auto-configured compliance
Manually create SSO                    SSO configured automatically

Takes days → Takes ~1 hour
```

### Control Tower Setup

```
Console → AWS Control Tower → Set up landing zone

What gets created automatically:
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│  Organization (Root)                                         │
│  ├── OU: Security                                           │
│  │   ├── Account: Audit (for security team)                 │
│  │   └── Account: Log Archive (centralized logs)            │
│  ├── OU: Sandbox (for experimentation)                      │
│  │                                                           │
│  Guardrails enabled:                                        │
│  ├── Preventive: Disallow public S3 buckets                │
│  ├── Preventive: Require MFA for root user                 │
│  ├── Detective: Detect unencrypted EBS volumes             │
│  ├── Detective: Detect public RDS instances                │
│  └── ...20+ guardrails                                     │
│                                                               │
│  Also configured:                                            │
│  ├── CloudTrail (organization-wide)                         │
│  ├── AWS Config (all accounts)                              │
│  ├── IAM Identity Center                                    │
│  ├── VPC with defaults for new accounts                     │
│  └── Account Factory (self-service account creation)        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Account Factory

```
Console → Control Tower → Account Factory → Create account

┌─────────────────────────────────────────────────────────┐
│ Account email:    [new-account@company.com]              │
│ Display name:     [Prod-NewService]                     │
│ SSO user email:   [admin@company.com]                   │
│ SSO user name:    [Admin]                               │
│ Organizational unit: [Production]                       │
│                                                         │
│ Account configuration:                                  │
│ ├── VPC CIDR: [10.0.0.0/16]                           │
│ ├── Region: [ap-south-1]                              │
│ ├── Subnets: [public + private per AZ]                │
│ └── Apply baseline guardrails: [Yes]                   │
│                                                         │
│ [Create account]                                        │
│                                                         │
│ ⏱️ Takes ~30 minutes to provision                       │
│ ✅ Account ready with networking, guardrails, SSO access │
└─────────────────────────────────────────────────────────┘
```

---

## Part 9: Real-World Company Scenarios

### Scenario 1: Startup (5-10 developers)

```
Organization:
├── Management Account (billing only)
├── OU: Workloads
│   ├── Account: Production
│   └── Account: Development (also used for staging)
└── OU: Sandbox
    └── Account: Sandbox (experiments)

SSO Setup:
├── Group: Founders → Admin on all accounts
├── Group: Developers → PowerUser on Dev, ReadOnly on Prod
└── Budget alarm: $500/month across all accounts

Networking:
├── Prod: Single VPC, 3 AZs, public + private subnets
└── Dev: Single VPC, single AZ (cost saving)
```

### Scenario 2: Mid-Size Company (50-100 developers)

```
Organization (with Control Tower):
├── Management Account (billing, org management)
├── OU: Security
│   ├── Account: Security-Audit (GuardDuty, Security Hub)
│   └── Account: Log-Archive (all logs centralized)
├── OU: Infrastructure
│   ├── Account: Network-Hub (Transit Gateway, DNS)
│   └── Account: Shared-Services (CI/CD, ECR, tools)
├── OU: Production
│   ├── Account: Prod-Team-A
│   ├── Account: Prod-Team-B
│   └── Account: Prod-Data
├── OU: Non-Production
│   ├── Account: Staging
│   └── Account: Development
└── OU: Sandbox
    └── Account: Sandbox (auto-nuke after 7 days)

SSO via External IdP (Okta/Azure AD):
├── Engineering → mapped to Developers group
├── SRE Team → mapped to DevOps group
├── Leadership → mapped to ReadOnly group
└── Security Team → mapped to SecurityAudit group
```

### Scenario 3: Enterprise (500+ developers)

```
Organization (Control Tower + custom):
├── Management Account
├── OU: Security & Governance
│   ├── Audit Account
│   ├── Log Archive Account
│   └── Security Tooling Account
├── OU: Infrastructure
│   ├── Network Account (Transit Gateway hub)
│   ├── DNS Account (Route 53 + Private Hosted Zones)
│   ├── Shared Services (CI/CD platform)
│   └── Backup Account (centralized backups)
├── OU: Business Unit A (BU-A)
│   ├── Prod accounts (per microservice or team)
│   ├── Staging accounts
│   └── Dev accounts
├── OU: Business Unit B (BU-B)
│   ├── Prod, Staging, Dev accounts...
├── OU: Data Platform
│   ├── Data Lake Account (S3 + Glue + Athena)
│   ├── Analytics Account (Redshift, QuickSight)
│   └── ML Account (SageMaker)
└── OU: Sandbox
    └── Individual developer accounts (auto-cleaned)

SCPs enforced:
├── Deny all non-approved regions
├── Deny disabling CloudTrail/Config/GuardDuty
├── Require encryption everywhere
├── Deny public S3 buckets in prod
├── Deny creating IAM users (force SSO only)
└── Deny leaving organization

Cost management:
├── AWS Budgets per OU and account
├── Cost anomaly detection enabled
├── RI/Savings Plans purchased at Org level (shared)
├── Chargeback reports per team using cost allocation tags
└── Monthly cost review meetings
```

---

## Part 10: Account Security Best Practices Summary

```
┌─────────────────────────────────────────────────────────────────┐
│              ACCOUNT SECURITY BEST PRACTICES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ROOT ACCOUNT:                                                   │
│  ├── ✅ Enable MFA (hardware key preferred)                      │
│  ├── ✅ No access keys for root user                            │
│  ├── ✅ Only use for tasks that REQUIRE root                    │
│  ├── ✅ Strong, unique password in password manager             │
│  └── ✅ Add alternate contacts (billing, security, operations)  │
│                                                                   │
│  ORGANIZATION:                                                   │
│  ├── ✅ Multi-account strategy (never everything in one account)│
│  ├── ✅ SCPs to restrict unused regions                         │
│  ├── ✅ SCPs to prevent disabling security services             │
│  ├── ✅ Dedicated Security OU with audit + log archive          │
│  └── ✅ Tag policies enforced at organization level             │
│                                                                   │
│  ACCESS:                                                         │
│  ├── ✅ Use IAM Identity Center (SSO) — no IAM users           │
│  ├── ✅ Federate with corporate IdP (Okta, Azure AD)           │
│  ├── ✅ MFA for all human users                                 │
│  ├── ✅ Temporary credentials only (no long-lived access keys) │
│  └── ✅ Least privilege — start with minimal, add as needed    │
│                                                                   │
│  MONITORING:                                                     │
│  ├── ✅ CloudTrail enabled (all regions, all accounts)          │
│  ├── ✅ GuardDuty enabled (all accounts, delegated admin)       │
│  ├── ✅ AWS Config enabled (compliance monitoring)              │
│  ├── ✅ Billing alarms configured                               │
│  └── ✅ Security Hub for centralized findings                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: Console Navigation

| Task | Console Path |
|------|-------------|
| View Organization | Console → AWS Organizations |
| Manage OUs | Organizations → Organize accounts |
| Create SCP | Organizations → Policies → Service control policies |
| Enable Identity Center | Console → IAM Identity Center |
| Create Users (SSO) | IAM Identity Center → Users |
| Assign Permissions | IAM Identity Center → AWS accounts → Assign |
| View Billing | Console → Billing Dashboard |
| Create Budget | Billing → Budgets → Create budget |
| Control Tower | Console → AWS Control Tower |
| Account Factory | Control Tower → Account Factory |

---

## What's Next?

In the next chapter, we'll dive deep into the resource hierarchy, tagging strategies, and how to organize resources within an account using tags and resource groups.

→ Next: [Chapter 3: Resource Hierarchy & Management](03-resource-hierarchy.md)

---

*Last Updated: May 2026*
