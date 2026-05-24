# Chapter 2: Account Setup & Tenant Configuration (Azure)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Creating a Personal Azure Account (Free Tier)](#part-1-creating-a-personal-azure-account-free-tier)
- [Part 2: Understanding Tenants (Microsoft Entra ID)](#part-2-understanding-tenants-microsoft-entra-id)
- [Part 3: Understanding Subscriptions](#part-3-understanding-subscriptions)
- [Part 4: Management Groups (Hierarchy)](#part-4-management-groups-hierarchy)
- [Part 5: Azure Policy (Guardrails)](#part-5-azure-policy-guardrails)
- [Part 6: Azure Landing Zones (Enterprise Best Practice)](#part-6-azure-landing-zones-enterprise-best-practice)
- [Part 7: Enterprise Agreement (EA) vs Pay-As-You-Go](#part-7-enterprise-agreement-ea-vs-pay-as-you-go)
- [Part 8: Cost Management & Budgets (Detailed)](#part-8-cost-management--budgets-detailed)
- [Part 9: Real-World Company Scenarios](#part-9-real-world-company-scenarios)
- [Part 10: Account Security Best Practices Summary](#part-10-account-security-best-practices-summary)
- [Quick Reference: Portal Navigation](#quick-reference-portal-navigation)
- [What's Next?](#whats-next)

---

## Overview

This chapter covers creating and managing Azure accounts — from a personal free account for learning, to a full enterprise setup with Microsoft Entra ID tenants, management groups, subscriptions, and governance.

```
What you'll learn:
├── Creating your personal Azure account (free tier + $200 credit)
├── Understanding Tenants (Microsoft Entra ID)
├── Subscriptions (billing + resource boundary)
├── Management Groups (hierarchy)
├── Azure Policy (guardrails)
├── Azure Landing Zones (enterprise best practice)
├── Cost Management & Budgets
├── How companies actually set this up
└── Step-by-step portal walkthrough
```

---

## Part 1: Creating a Personal Azure Account (Free Tier)

### What You Need Before Starting

| Requirement | Details |
|-------------|---------|
| Microsoft account | Outlook/Hotmail or any email (creates Microsoft account) |
| Phone number | For verification |
| Credit/Debit card | Required (won't be charged during free period) |
| Valid ID | Government ID may be needed for verification |

### Step-by-Step: Creating a Free Account

```
Portal: https://azure.microsoft.com/free → "Start free"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: Sign In / Create Microsoft Account
┌─────────────────────────────────────────────────────────┐
│ Sign in with existing Microsoft account                 │
│ OR                                                      │
│ "Create one!" → New Microsoft account                   │
│                                                         │
│ Email: [your-email@outlook.com / any email]             │
│ Password: [Strong password]                             │
│                                                         │
│ 💡 For personal: Any email works                        │
│ 💡 For company: Use work email (company@domain.com)     │
└─────────────────────────────────────────────────────────┘

Step 2: About You
┌─────────────────────────────────────────────────────────┐
│ Country/Region:  [India]                                │
│ First name:      [Your First Name]                      │
│ Last name:       [Your Last Name]                       │
│ Email address:   [your-email@domain.com]                │
│ Phone:           [+91-XXXXXXXXXX]                       │
│                                                         │
│ → Phone verification (SMS code)                         │
└─────────────────────────────────────────────────────────┘

Step 3: Identity Verification
┌─────────────────────────────────────────────────────────┐
│ Verification by: ○ Credit Card  ○ Debit Card            │
│                                                         │
│ Card number:     [XXXX-XXXX-XXXX-XXXX]                 │
│ Expiry:          [MM/YY]                                │
│ CVV:             [XXX]                                   │
│ Cardholder:      [Name on Card]                         │
│ Address:         [Your Billing Address]                  │
│                                                         │
│ ⚠️ Temporary hold of ~$1/₹2 for verification            │
│ 💡 You will NOT be charged until you upgrade             │
│ 💡 Account auto-suspends after 30 days / $200 spent     │
└─────────────────────────────────────────────────────────┘

Step 4: Agreement
┌─────────────────────────────────────────────────────────┐
│ [✓] I agree to the subscription agreement, offer       │
│     details, and privacy statement                      │
│                                                         │
│ [Sign up]                                               │
└─────────────────────────────────────────────────────────┘

✅ Account Created! Redirected to Azure Portal.

What you get:
├── $200 free credit for 30 days
├── 12 months of popular free services
├── 55+ always-free services
├── Default subscription: "Azure subscription 1"
├── Default tenant: yourname.onmicrosoft.com
└── Default directory: Default Directory
```

### Azure Free Account Details

```
┌─────────────────────────────────────────────────────────────┐
│                   AZURE FREE ACCOUNT TIERS                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Tier 1: $200 Credit (First 30 days)                        │
│  ├── Use ANY Azure service                                  │
│  ├── No restrictions on services                            │
│  ├── Expires after 30 days OR when credit runs out          │
│  └── Must upgrade to pay-as-you-go to continue              │
│                                                               │
│  Tier 2: 12-Month Free Services                              │
│  ├── Linux B1s VM: 750 hours/month                          │
│  ├── Windows B1s VM: 750 hours/month                        │
│  ├── Managed Disks: 2x 64 GB P6 SSD                        │
│  ├── Blob Storage: 5 GB LRS                                 │
│  ├── File Storage: 5 GB LRS                                 │
│  ├── SQL Database: 250 GB S0                                │
│  ├── Cosmos DB: 25 GB + 1000 RU/s                           │
│  ├── Bandwidth: 15 GB outbound                              │
│  └── ... more services                                       │
│                                                               │
│  Tier 3: Always Free (No expiration)                         │
│  ├── App Service: 10 web apps (F1 tier)                     │
│  ├── Functions: 1M executions/month                         │
│  ├── Event Grid: 100K operations/month                      │
│  ├── Azure DevOps: 5 users + unlimited repos                │
│  ├── Azure Active Directory: Basic features                  │
│  ├── Service Bus: 12.5K messaging operations                │
│  ├── Azure Advisor: Always free                              │
│  ├── Azure Monitor: Basic metrics free                       │
│  └── 55+ more services with free quotas                     │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Immediately After Account Creation (Setup Checklist)

```
⚠️  DAY 1 SETUP TASKS (Do these IMMEDIATELY):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

□ 1. Enable MFA on your account
     Portal → Microsoft Entra ID → Users → Your user →
     Authentication methods → Add method → Microsoft Authenticator
     
□ 2. Review your subscription
     Portal → Subscriptions → "Azure subscription 1"
     Note: Subscription ID (you'll need this)
     
□ 3. Create a Budget Alert
     Portal → Cost Management → Budgets → Add
     - Name: "Monthly Limit"
     - Reset period: Monthly
     - Amount: $20 (or your limit)
     - Alert at: 50%, 80%, 100%
     - Recipients: your email

□ 4. Set up Azure Cloud Shell
     Click [>_] icon in top bar
     Choose: Bash or PowerShell
     Creates: Storage account for shell state (costs ~$0.01/month)
     
□ 5. Review default resource providers
     Portal → Subscriptions → Your sub → Resource providers
     Most needed ones are already registered
     
□ 6. Familiarize with the portal
     - Pin services to favorites (left sidebar)
     - Try the search bar (searches everything!)
     - Check Azure Advisor (free recommendations)
```

---

## Part 2: Understanding Tenants (Microsoft Entra ID)

### What is a Tenant?

A **Tenant** is a dedicated instance of Microsoft Entra ID (formerly Azure Active Directory) that represents your organization. It's the identity foundation for everything in Azure.

```
┌─────────────────────────────────────────────────────────────────┐
│                     MICROSOFT ENTRA ID TENANT                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Tenant = Your organization's identity container                 │
│                                                                   │
│  Contains:                                                       │
│  ├── Users (employees, guests, service principals)              │
│  ├── Groups (security groups, Microsoft 365 groups)             │
│  ├── App Registrations (applications)                           │
│  ├── Enterprise Applications (SSO/SAML apps)                    │
│  ├── Roles & Permissions (directory-level)                      │
│  ├── Conditional Access Policies                                │
│  ├── Device registrations                                       │
│  └── Tenant settings (branding, external access)                │
│                                                                   │
│  Every tenant has:                                               │
│  ├── Tenant ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (GUID)    │
│  ├── Primary domain: yourcompany.onmicrosoft.com (default)      │
│  ├── Custom domains: yourcompany.com (verified)                 │
│  └── Global Administrator(s): Full control of tenant           │
│                                                                   │
│  Relationship:                                                   │
│  ┌──────────────┐       ┌──────────────────┐                   │
│  │ Entra ID     │ 1───N │ Azure            │                   │
│  │ Tenant       │───────│ Subscriptions    │                   │
│  │              │       │                  │                   │
│  │ (Identity)   │       │ (Resources)      │                   │
│  └──────────────┘       └──────────────────┘                   │
│                                                                   │
│  One Tenant → Many Subscriptions                                 │
│  One Subscription → Exactly One Tenant                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### When Do You Get a Tenant?

```
Scenario 1: Personal Free Account
─────────────────────────────────
When you sign up → A new tenant is auto-created
Tenant domain: yourname.onmicrosoft.com
You are: Global Administrator of this tenant
Contains: Just you (1 user)

Scenario 2: Company Already Has Microsoft 365
─────────────────────────────────────────────
Your company already has a tenant (from Office 365/Microsoft 365)
Tenant domain: company.onmicrosoft.com + company.com (custom)
Contains: All company users, groups, devices
Azure subscriptions link to THIS tenant

Scenario 3: Creating a New Company Tenant
─────────────────────────────────────────
Azure Portal → Microsoft Entra ID → Manage Tenants → Create
Use when: Starting fresh company setup, or need separate tenant
```

### Tenant vs Organization (Cross-Cloud Comparison)

```
┌──────────────────────────────────────────────────────────┐
│  Azure TENANT ≈ GCP ORGANIZATION ≈ AWS ORGANIZATION      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Azure:                                                  │
│  └── Tenant (Entra ID) = Identity + Top-level container │
│      └── Management Groups → Subscriptions → RGs        │
│                                                          │
│  GCP:                                                    │
│  └── Organization (Cloud Identity) = Identity + Top     │
│      └── Folders → Projects → Resources                 │
│                                                          │
│  AWS:                                                    │
│  └── Organization (Management Account) = Top-level      │
│      └── OUs → Accounts → Resources                    │
│      └── IAM = Identity (separate from Org structure)   │
│                                                          │
│  Key difference: In Azure, identity (Entra ID) is       │
│  SEPARATE from resource hierarchy (Management Groups).  │
│  They're linked but managed independently.              │
└──────────────────────────────────────────────────────────┘
```

---

## Part 3: Understanding Subscriptions

### What is a Subscription?

A **Subscription** is the billing container and logical boundary for Azure resources. Think of it as your "account" for deploying things.

```
┌─────────────────────────────────────────────────────────────────┐
│                     AZURE SUBSCRIPTION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  A subscription provides:                                        │
│  ├── BILLING BOUNDARY — All resources billed to this sub        │
│  ├── ACCESS BOUNDARY — RBAC can be scoped to subscription       │
│  ├── POLICY BOUNDARY — Azure Policy can target subscriptions    │
│  ├── RESOURCE LIMITS — Quotas per subscription                  │
│  │   (e.g., max 250 storage accounts, max 25K vCPUs per region)│
│  └── TRUST RELATIONSHIP — Linked to exactly one Entra ID tenant │
│                                                                   │
│  Subscription Types:                                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Type                │ Use Case                              │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │ Free                │ Learning ($200 credit, 30 days)       │ │
│  │ Pay-As-You-Go      │ Default for most (credit card)         │ │
│  │ Visual Studio       │ Developers (monthly credit $50-$150) │ │
│  │ Enterprise Agreement│ Large companies (annual commitment)   │ │
│  │ CSP (Cloud Solution)│ Through Microsoft Partner             │ │
│  │ Dev/Test            │ Discounted for dev (no Windows fee)   │ │
│  │ Sponsorship         │ Microsoft-provided credits            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating Additional Subscriptions

```
Portal → Subscriptions → Add

┌─────────────────────────────────────────────────────────┐
│                  CREATE SUBSCRIPTION                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Offer:                                                  │
│ ├── Pay-As-You-Go (most common)                        │
│ ├── Dev/Test (if EA customer)                          │
│ └── Other (depends on agreement)                       │
│                                                         │
│ Subscription name: [Prod-WebApp]                        │
│ (Can be changed later)                                  │
│                                                         │
│ Billing account: [Your billing account]                 │
│ (EA enrollment or pay-as-you-go credit card)           │
│                                                         │
│ Directory: [yourcompany.onmicrosoft.com]                │
│ (Which Entra ID tenant to associate with)              │
│                                                         │
│ Management group: [Production] or [Tenant Root Group]   │
│ (Where in hierarchy to place it)                       │
│                                                         │
│ [Create]                                                │
│                                                         │
│ 💡 Company naming convention:                            │
│    <company>-<environment>-<purpose>                    │
│    Example: techcorp-prod-webapp                        │
│    Example: techcorp-dev-all                           │
└─────────────────────────────────────────────────────────┘

CLI:
  az account create \
    --offer-type MS-AZR-0003P \
    --display-name "Prod-WebApp" \
    --enrollment-account-name "ENROLLMENT_ACCOUNT_ID"
```

### Multiple Subscriptions Strategy

```
Why multiple subscriptions?

┌─────────────────────────────────────────────────────────────────┐
│ Reason                    │ Example                              │
├─────────────────────────────────────────────────────────────────┤
│ Billing separation       │ Prod vs Dev costs tracked separately │
│ Access isolation          │ Dev team can't touch prod resources  │
│ Resource limits           │ Each sub has its own quotas          │
│ Policy isolation          │ Different policies per environment   │
│ Compliance               │ Sensitive data in restricted sub     │
│ Organizational structure │ Per team or per business unit        │
└─────────────────────────────────────────────────────────────────┘

Common patterns:
├── Single sub (small team/startup)
├── Per environment (prod, staging, dev) ← Most common start
├── Per team + environment (team-a-prod, team-a-dev, team-b-prod)
└── Per workload (specific compliance requirements)
```

---

## Part 4: Management Groups (Hierarchy)

### What are Management Groups?

Management Groups are containers that help you manage access, policy, and compliance for **multiple subscriptions**. They form a hierarchy above subscriptions.

```
┌─────────────────────────────────────────────────────────────────┐
│                   MANAGEMENT GROUP HIERARCHY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Tenant Root Group (auto-created, tied to Entra ID tenant)      │
│  │                                                               │
│  ├── Management Group: Platform                                 │
│  │   ├── MG: Identity                                          │
│  │   │   └── Sub: Identity-Prod                                │
│  │   ├── MG: Management                                        │
│  │   │   └── Sub: Management-Prod                              │
│  │   └── MG: Connectivity                                      │
│  │       └── Sub: Connectivity-Prod                            │
│  │                                                               │
│  ├── Management Group: Landing Zones                            │
│  │   ├── MG: Production                                        │
│  │   │   ├── Sub: Prod-App1                                   │
│  │   │   └── Sub: Prod-App2                                   │
│  │   ├── MG: Non-Production                                    │
│  │   │   ├── Sub: Staging                                     │
│  │   │   └── Sub: Development                                 │
│  │   └── MG: Corp (internal apps)                              │
│  │       └── Sub: Corp-Tools                                   │
│  │                                                               │
│  ├── Management Group: Sandbox                                  │
│  │   └── Sub: Sandbox-Developers                               │
│  │                                                               │
│  └── Management Group: Decommissioned                           │
│      └── Sub: Old-Project (locked down)                        │
│                                                                   │
│  KEY RULES:                                                      │
│  ├── Max 6 levels deep (excluding Root and Subscription level)  │
│  ├── Each sub belongs to exactly ONE management group           │
│  ├── RBAC inherits: MG → child MG → Subscription → RG → Resource│
│  ├── Azure Policy inherits the same way                         │
│  └── 10,000 management groups per tenant max                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating Management Groups

```
Portal → Management Groups → Add management group

┌─────────────────────────────────────────────────────────┐
│ Management group ID:     [mg-production]                │
│ (Cannot change after creation! Use meaningful ID)       │
│                                                         │
│ Management group name:   [Production]                   │
│ (Display name, can be changed)                          │
│                                                         │
│ Parent: [Tenant Root Group] or [Another MG]             │
│                                                         │
│ [Submit]                                                │
└─────────────────────────────────────────────────────────┘

CLI:
  az account management-group create \
    --name "mg-production" \
    --display-name "Production" \
    --parent "mg-landing-zones"

Move subscription into MG:
  Portal → Management Groups → select MG → Add subscription
  
  az account management-group subscription add \
    --name "mg-production" \
    --subscription "SUBSCRIPTION_ID"
```

### Inheritance Model

```
RBAC + Policy Inheritance:

  Tenant Root Group
  ├── Policy: "Audit VMs without managed disks" → Applied to ALL below
  ├── RBAC: "Security Reader" → security-team@company.com
  │
  ├── MG: Production
  │   ├── Policy: "Require tag: Environment" → All prod subscriptions
  │   ├── RBAC: "Contributor" → devops-team@company.com
  │   │
  │   └── Sub: Prod-App1
  │       ├── Policy: "Allowed VM sizes" → Only D-series + E-series
  │       ├── RBAC: "Reader" → developers@company.com
  │       │
  │       └── RG: rg-prod-compute
  │           ├── RBAC: "VM Contributor" → specific-dev@company.com
  │           │
  │           └── VM: prod-web-01
  │               Effective policies: ALL from above (merged)
  │               Effective RBAC: ALL from above (union)
  │
  │   Effective permissions on VM "prod-web-01":
  │   ├── security-team: Security Reader (from Root)
  │   ├── devops-team: Contributor (from MG:Production)
  │   ├── developers: Reader (from Subscription)
  │   └── specific-dev: VM Contributor (from Resource Group)
```

---

## Part 5: Azure Policy (Guardrails)

### What is Azure Policy?

Azure Policy enforces organizational standards and assesses compliance at scale. Think of it as "what IS allowed" vs "what is NOT allowed" for your Azure resources.

```
Azure Policy vs RBAC:

  RBAC: "WHO can do WHAT" (identity-based)
  └── User X has Contributor role = can create/modify resources

  Azure Policy: "WHAT configurations are ALLOWED" (resource-based)  
  └── Only D-series VMs allowed, must have tags, must use managed disks
  
  Both work together:
  ├── RBAC allows the user to create a VM
  └── Policy checks: Is the VM configuration compliant?
      ├── YES → VM created
      └── NO → VM creation DENIED (or flagged non-compliant)
```

### Policy Effects

```
┌──────────────────────────────────────────────────────────────────┐
│                      POLICY EFFECTS                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DENY          → Block the action completely                      │
│                  "You cannot create a VM without these tags"       │
│                                                                    │
│  AUDIT         → Allow but flag as non-compliant                  │
│                  "This VM doesn't have tags (warning)"            │
│                                                                    │
│  MODIFY        → Auto-fix the resource during create/update       │
│                  "Add default tags automatically"                  │
│                                                                    │
│  APPEND        → Add fields to resources during create/update     │
│                  "Add diagnostic settings to all resources"       │
│                                                                    │
│  DEPLOYIFNOTEXISTS → Deploy additional resource if missing        │
│                      "Deploy Microsoft Defender agent on all VMs" │
│                                                                    │
│  AUDITIFNOTEXISTS  → Audit if related resource is missing         │
│                      "Flag VMs without backup configured"         │
│                                                                    │
│  DISABLED     → Policy exists but is not enforced                 │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Creating and Assigning Policies

```
Portal → Policy → Definitions → (browse built-in or create custom)

BUILT-IN POLICIES (hundreds available):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Common ones:
├── "Allowed locations" → Restrict which regions resources can be in
├── "Allowed virtual machine size SKUs" → Restrict VM sizes
├── "Require a tag and its value on resources" → Enforce tagging
├── "Storage accounts should restrict network access" → Security
├── "Audit VMs that do not use managed disks" → Compliance
├── "Deploy Diagnostic Settings for all resources" → Monitoring
├── "Kubernetes cluster should not allow privileged containers" → Security
└── "SQL servers should have auditing enabled" → Compliance

ASSIGNING A POLICY:
━━━━━━━━━━━━━━━━━━━
Portal → Policy → Assignments → Assign policy

┌─────────────────────────────────────────────────────────┐
│                    ASSIGN POLICY                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Scope:        [Management Group / Subscription / RG]    │
│               ├── /mg-production (applies to all below)│
│               ├── /subscriptions/xxx (one subscription)│
│               └── /subscriptions/xxx/rg-name (one RG)  │
│                                                         │
│ Exclusions:   [Optional - exclude specific RGs/subs]    │
│               e.g., Exclude sandbox resource group      │
│                                                         │
│ Policy:       [Allowed locations]                       │
│                                                         │
│ Parameters:                                             │
│ ├── Allowed locations: [✓ East US, ✓ Central India,    │
│ │                       ✓ West Europe]                  │
│ │   (only these regions allowed for resources)         │
│                                                         │
│ Enforcement:  ○ Enabled (deny non-compliant)           │
│               ○ Disabled (audit only, don't block)     │
│                                                         │
│ Non-compliance message:                                 │
│ "Resources must be deployed in approved regions only.  │
│  Contact cloud-team@company.com for exceptions."       │
│                                                         │
│ Remediation: (for DeployIfNotExists/Modify)            │
│ [✓] Create remediation task                            │
│ [✓] Create managed identity for remediation            │
│                                                         │
│ [Create]                                                │
└─────────────────────────────────────────────────────────┘
```

### Policy Initiatives (Policy Sets)

```
An INITIATIVE groups multiple policies together for easy assignment.

Portal → Policy → Definitions → Initiative definition → Create

Example: "Company Security Baseline" initiative:
┌─────────────────────────────────────────────────────────┐
│ Initiative: Company Security Baseline                    │
│                                                         │
│ Included policies:                                      │
│ ├── Allowed locations (only approved regions)           │
│ ├── Require tag: "Environment" on all resources         │
│ ├── Require tag: "CostCenter" on all resources          │
│ ├── Storage accounts must use HTTPS                    │
│ ├── SQL databases must have TDE enabled                │
│ ├── VMs must use managed disks                         │
│ ├── Network interfaces should not have public IPs      │
│ ├── Key Vault should have soft delete enabled          │
│ └── Audit VMs without endpoint protection              │
│                                                         │
│ Assign this ONE initiative instead of 9 separate        │
│ policy assignments!                                     │
└─────────────────────────────────────────────────────────┘

Built-in initiatives:
├── "Azure Security Benchmark" (comprehensive security)
├── "CIS Microsoft Azure Foundations Benchmark"
├── "NIST SP 800-53"
├── "ISO 27001"
├── "PCI DSS"
└── "HIPAA/HITRUST"
```

---

## Part 6: Azure Landing Zones (Enterprise Best Practice)

### What is a Landing Zone?

An Azure Landing Zone is a pre-configured, secure, multi-subscription Azure environment following Microsoft's best practices. It's the recommended starting point for enterprise Azure adoption.

```
┌─────────────────────────────────────────────────────────────────┐
│              AZURE LANDING ZONE ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Tenant Root Group                                               │
│  │                                                               │
│  ├── MG: Platform ──────────────────────────────────────────    │
│  │   │  (Core infrastructure managed by platform team)          │
│  │   │                                                           │
│  │   ├── MG: Identity                                           │
│  │   │   └── Sub: Identity                                     │
│  │   │       └── Entra ID Connect, Domain Controllers          │
│  │   │                                                           │
│  │   ├── MG: Management                                        │
│  │   │   └── Sub: Management                                   │
│  │   │       ├── Log Analytics Workspace (central)             │
│  │   │       ├── Automation Account                            │
│  │   │       └── Azure Monitor (central dashboards)            │
│  │   │                                                           │
│  │   └── MG: Connectivity                                      │
│  │       └── Sub: Connectivity                                 │
│  │           ├── Hub VNet (10.0.0.0/16)                        │
│  │           │   ├── Azure Firewall                            │
│  │           │   ├── VPN Gateway                               │
│  │           │   ├── ExpressRoute Gateway                      │
│  │           │   └── Azure Bastion                             │
│  │           ├── Private DNS Zones                              │
│  │           └── DDoS Protection Plan                          │
│  │                                                               │
│  ├── MG: Landing Zones ──────────────────────────────────────   │
│  │   │  (Where workloads live — application teams deploy here) │
│  │   │                                                           │
│  │   ├── MG: Production (Corp + Online)                        │
│  │   │   ├── Sub: Prod-App-Team-A                             │
│  │   │   │   ├── Spoke VNet (10.1.0.0/16) peered to Hub      │
│  │   │   │   ├── RG: rg-app-a-compute (AKS/VMs)              │
│  │   │   │   └── RG: rg-app-a-data (SQL/Storage)             │
│  │   │   └── Sub: Prod-App-Team-B                             │
│  │   │       └── Spoke VNet (10.2.0.0/16) peered to Hub      │
│  │   │                                                           │
│  │   └── MG: Non-Production                                    │
│  │       ├── Sub: Dev-All                                      │
│  │       └── Sub: Staging-All                                  │
│  │                                                               │
│  ├── MG: Sandbox ────────────────────────────────────────────   │
│  │   └── Sub: Sandbox (relaxed policies, budget capped)        │
│  │                                                               │
│  └── MG: Decommissioned ────────────────────────────────────   │
│      └── Sub: Old-Projects (deny all new resources)            │
│                                                                   │
│  Networking Pattern: HUB-AND-SPOKE                              │
│  ┌─────────────────────────────────────────────┐               │
│  │          Hub VNet (Connectivity Sub)         │               │
│  │    ┌──────────────────────────────────┐     │               │
│  │    │ Azure Firewall  │  VPN Gateway   │     │               │
│  │    │ DNS Resolver    │  Bastion       │     │               │
│  │    └───────┬─────────┴───────┬────────┘     │               │
│  └────────────┼─────────────────┼──────────────┘               │
│               │ VNet Peering    │ VNet Peering                  │
│        ┌──────┘                 └──────┐                        │
│        ▼                               ▼                        │
│  ┌──────────────┐             ┌──────────────┐                 │
│  │ Spoke VNet A │             │ Spoke VNet B │                 │
│  │ (Team A Prod)│             │ (Team B Prod)│                 │
│  │ 10.1.0.0/16  │             │ 10.2.0.0/16  │                 │
│  └──────────────┘             └──────────────┘                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deploying a Landing Zone

```
Two approaches:

1. Azure Landing Zone Accelerator (Portal-based)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Portal → Deploy a custom template → 
"Deploy Azure landing zone" (Microsoft provided)

Wizard walks you through:
├── Platform management group structure
├── Connectivity (Hub VNet + Firewall + VPN/ER)
├── Identity (AD Connect)
├── Logging (Log Analytics)
├── Security (Defender for Cloud)
└── Landing zone subscriptions

2. Terraform/Bicep Modules (IaC — Recommended for production)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GitHub: Azure/terraform-azurerm-caf-enterprise-scale
GitHub: Azure/ALZ-Bicep

Advantages:
├── Version-controlled infrastructure
├── Repeatable and testable
├── PR-based changes with peer review
└── Drift detection
```

---

## Part 7: Enterprise Agreement (EA) vs Pay-As-You-Go

```
┌──────────────────────────────────────────────────────────────────┐
│              BILLING MODELS COMPARISON                             │
├──────────────┬────────────────────┬──────────────────────────────┤
│              │ Pay-As-You-Go      │ Enterprise Agreement (EA)    │
├──────────────┼────────────────────┼──────────────────────────────┤
│ Payment      │ Credit card        │ Annual commitment (prepaid)  │
│ Commitment   │ None               │ 1-3 years, minimum spend     │
│ Discount     │ None (list price)  │ Negotiated discount          │
│ Billing      │ Monthly invoice    │ Annual + overage quarterly   │
│ Support      │ Pay separately     │ Often included               │
│ Subscriptions│ Self-service       │ EA Portal + self-service     │
│ Best for     │ Small/medium       │ Large enterprises            │
│ Dev/Test     │ Regular pricing    │ Special Dev/Test rates        │
│ Management   │ Portal only        │ EA Portal + Azure Portal     │
└──────────────┴────────────────────┴──────────────────────────────┘

EA Hierarchy:
Enterprise Agreement
├── Department (grouping for billing) ← Optional
│   ├── Account (EA enrollment account)
│   │   ├── Subscription: Prod-App1
│   │   └── Subscription: Prod-App2
│   └── Account
│       └── Subscription: Dev-All
└── Department
    └── Account
        └── Subscription: Data-Platform
```

---

## Part 8: Cost Management & Budgets (Detailed)

### Cost Management Portal

```
Portal → Cost Management + Billing

┌─────────────────────────────────────────────────────────────┐
│                   COST MANAGEMENT                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Key sections:                                               │
│                                                               │
│  1. COST ANALYSIS                                           │
│     └── Visualize and analyze spending                      │
│         ├── By resource group                               │
│         ├── By service/meter                                │
│         ├── By tag                                          │
│         ├── By location                                     │
│         ├── Daily/monthly trends                            │
│         └── Forecast future spend                           │
│                                                               │
│  2. BUDGETS                                                  │
│     └── Set spending limits with alerts                     │
│                                                               │
│  3. ADVISOR RECOMMENDATIONS                                  │
│     └── Right-sizing, reserved instances, unused resources  │
│                                                               │
│  4. COST ALERTS                                              │
│     └── Anomaly detection, budget alerts                    │
│                                                               │
│  5. EXPORTS                                                  │
│     └── Schedule automatic cost data export to storage      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Creating a Detailed Budget

```
Portal → Cost Management → Budgets → Add

┌─────────────────────────────────────────────────────────┐
│                    CREATE BUDGET                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Scope: [Subscription / Resource Group / MG]             │
│                                                         │
│ Budget details:                                         │
│ ├── Name: [prod-monthly-budget]                        │
│ ├── Reset period: [Monthly / Quarterly / Annual]       │
│ ├── Creation date: [auto]                              │
│ ├── Expiration date: [optional]                        │
│ └── Amount: [$5,000]                                   │
│                                                         │
│ Filters (optional - refine what's tracked):            │
│ ├── Resource groups: [rg-prod-compute ✓]               │
│ ├── Services: [Virtual Machines ✓] [SQL Database ✓]    │
│ ├── Tags: [Environment: Production]                    │
│ └── Locations: [Central India]                         │
│                                                         │
│ Alert conditions:                                       │
│ ┌───────────────────────────────────────────────────┐  │
│ │ Type      │ % of Budget │ Action                  │  │
│ ├───────────────────────────────────────────────────┤  │
│ │ Actual    │ 50%         │ Email: team@company.com │  │
│ │ Actual    │ 75%         │ Email: lead@company.com │  │
│ │ Actual    │ 90%         │ Email: manager@company  │  │
│ │ Actual    │ 100%        │ Email + Action Group    │  │
│ │ Forecasted│ 110%        │ Email: cfo@company.com  │  │
│ └───────────────────────────────────────────────────┘  │
│                                                         │
│ Action Group (optional):                                │
│ ├── Send email                                         │
│ ├── Send SMS                                           │
│ ├── Trigger Azure Function (auto-shutdown!)            │
│ ├── Trigger Logic App (create ticket in ServiceNow)    │
│ └── Call webhook                                       │
│                                                         │
│ [Create]                                                │
└─────────────────────────────────────────────────────────┘
```

---

## Part 9: Real-World Company Scenarios

### Scenario 1: Startup (5-10 developers)

```
Tenant: startup.onmicrosoft.com (+ startup.com custom domain)
│
├── Management Group: Root
│   └── Management Group: Workloads
│       ├── Sub: Production (pay-as-you-go)
│       │   ├── RG: rg-prod-app
│       │   └── RG: rg-prod-data
│       ├── Sub: Development (pay-as-you-go)
│       │   └── RG: rg-dev
│       └── Sub: Sandbox (spending cap: $100/month)

Security:
├── All users in Entra ID with MFA enforced
├── Conditional Access: Require MFA for Azure Portal
├── Azure Policy: Allowed locations (1-2 regions)
├── Budget alerts: $200/month on prod, $50/month on dev
└── Azure Defender: Free tier enabled
```

### Scenario 2: Mid-Size Company (50-100 developers)

```
Tenant: techcorp.onmicrosoft.com (+ techcorp.com)
Microsoft 365 + Azure (same tenant)
│
├── MG: Platform
│   ├── Sub: Connectivity → Hub VNet, Azure Firewall, VPN
│   └── Sub: Management → Log Analytics, Automation
│
├── MG: Production
│   ├── Sub: Prod-Frontend → App Service, Front Door, CDN
│   ├── Sub: Prod-Backend → AKS cluster, APIs
│   └── Sub: Prod-Data → Azure SQL, Cosmos DB, Storage
│
├── MG: Non-Production
│   ├── Sub: Staging → Mirror of prod (smaller scale)
│   └── Sub: Development → Free-form dev environment
│
└── MG: Sandbox
    └── Sub: Developer-Sandbox → Capped budget, relaxed policies

Azure Policy (at Root MG):
├── Allowed locations: East US, Central India, West Europe
├── Require tags: Environment, Owner, CostCenter
├── Deny public IP on VMs (except sandbox)
├── Require HTTPS on storage accounts
└── Deploy Defender for Cloud on all subscriptions

Azure DevOps: dev.azure.com/techcorp
├── Pipelines deploy to all environments
├── Approvals for prod deployments
└── Service connections per subscription (with managed identity)
```

### Scenario 3: Enterprise (500+ developers)

```
Tenant: enterprise.onmicrosoft.com (+ enterprise.com)
Enterprise Agreement: $1M/year committed
Hybrid: Azure + On-premises (ExpressRoute connected)
│
├── MG: Platform
│   ├── MG: Identity
│   │   └── Sub: Identity → Entra ID Connect, Domain Controllers
│   ├── MG: Management
│   │   └── Sub: Management → Log Analytics (10TB/day), SIEM
│   └── MG: Connectivity
│       └── Sub: Connectivity → Hub VNet, ExpressRoute (2 circuits),
│           Azure Firewall Premium, Private DNS, DDoS Protection
│
├── MG: Landing Zones
│   ├── MG: Corp (internal traffic through firewall)
│   │   ├── Sub: Corp-HR
│   │   ├── Sub: Corp-Finance
│   │   └── Sub: Corp-Internal-Tools
│   ├── MG: Online (internet-facing, through Front Door)
│   │   ├── Sub: Online-CustomerPortal
│   │   ├── Sub: Online-API-Platform
│   │   └── Sub: Online-Mobile-Backend
│   └── MG: Non-Production
│       ├── Sub: Staging-Corp
│       ├── Sub: Staging-Online
│       ├── Sub: Dev-Corp
│       └── Sub: Dev-Online
│
├── MG: Sandbox
│   └── Sub: Innovation-Lab (auto-delete resources after 14 days)
│
└── MG: Decommissioned
    └── Sub: Legacy-App (deny-all policy, read-only)

Security:
├── Entra ID P2 (PIM, Conditional Access, Identity Protection)
├── Privileged Identity Management (just-in-time admin access)
├── Conditional Access: Device compliance + MFA + Location
├── Microsoft Defender for Cloud (all plans enabled)
├── Microsoft Sentinel (SIEM/SOAR) in Management sub
├── Azure DDoS Protection Standard
└── Azure Policy: CIS Benchmark + Custom corporate policies

Cost Management:
├── EA with negotiated rates
├── Reserved Instances: 3-year for steady workloads
├── Azure Hybrid Benefit: Windows Server + SQL Server licenses
├── Dev/Test subscriptions: No Windows licensing fees
├── Budgets per department + automated alerts
├── Monthly cost review with FinOps team
└── Chargeback via tags: Department, CostCenter, Application
```

---

## Part 10: Account Security Best Practices Summary

```
┌─────────────────────────────────────────────────────────────────┐
│              AZURE ACCOUNT SECURITY BEST PRACTICES               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  IDENTITY (Entra ID):                                           │
│  ├── ✅ MFA for ALL users (Conditional Access or Security Defaults)│
│  ├── ✅ Conditional Access policies (location + device + risk)  │
│  ├── ✅ PIM for admin roles (just-in-time, not permanent)       │
│  ├── ✅ Break-glass accounts (2 emergency admin accounts)       │
│  │      → MFA with hardware key, excluded from Conditional Access│
│  │      → Monitor with alerts on usage                          │
│  ├── ✅ No permanent Global Admin assignments                   │
│  ├── ✅ Block legacy authentication protocols                   │
│  └── ✅ Regular access reviews                                  │
│                                                                   │
│  SUBSCRIPTION & GOVERNANCE:                                      │
│  ├── ✅ Management Group hierarchy (not flat)                   │
│  ├── ✅ Azure Policy at MG level (inherited)                    │
│  ├── ✅ Resource locks on critical resources (CanNotDelete)     │
│  ├── ✅ Tagging policy enforced (for cost tracking)             │
│  ├── ✅ Budget alerts on all subscriptions                      │
│  └── ✅ Deny policy for unused regions                          │
│                                                                   │
│  ACCESS CONTROL (RBAC):                                          │
│  ├── ✅ Use built-in roles where possible                       │
│  ├── ✅ Assign to GROUPS, not individual users                  │
│  ├── ✅ Scope as narrowly as possible (RG > Sub > MG)          │
│  ├── ✅ Use Managed Identities (not service principal secrets) │
│  └── ✅ Regular access review and cleanup                       │
│                                                                   │
│  MONITORING:                                                     │
│  ├── ✅ Diagnostic settings on all resources → Log Analytics    │
│  ├── ✅ Activity Log alerts for critical operations             │
│  ├── ✅ Azure Monitor alerts for infrastructure health          │
│  ├── ✅ Microsoft Defender for Cloud enabled                    │
│  └── ✅ Microsoft Sentinel for security monitoring (enterprise) │
│                                                                   │
│  NETWORKING:                                                     │
│  ├── ✅ Hub-spoke topology with Azure Firewall                  │
│  ├── ✅ No public IPs directly on VMs                          │
│  ├── ✅ Use Azure Bastion or Just-In-Time VM access            │
│  ├── ✅ Private Endpoints for PaaS services                    │
│  ├── ✅ NSG flow logs enabled                                   │
│  └── ✅ DDoS Protection for internet-facing workloads          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: Portal Navigation

| Task | Portal Path |
|------|------------|
| View/Switch Tenant | Portal → Top-right → Directory + Subscription |
| Manage Subscriptions | Portal → Subscriptions |
| Management Groups | Portal → Management Groups |
| Azure Policy | Portal → Policy |
| Create Budget | Portal → Cost Management → Budgets → Add |
| Cost Analysis | Portal → Cost Management → Cost analysis |
| Entra ID | Portal → Microsoft Entra ID |
| Users & Groups | Entra ID → Users / Groups |
| Conditional Access | Entra ID → Security → Conditional Access |
| RBAC | Any resource → Access Control (IAM) |
| Activity Log | Any resource → Activity Log |

---

## What's Next?

In the next chapter, we'll dive into the resource hierarchy in detail — Resource Groups, resource naming conventions, tagging strategy, and resource locks.

→ Next: [Chapter 3: Resource Hierarchy & Management](03-resource-hierarchy.md)

---

*Last Updated: May 2026*
