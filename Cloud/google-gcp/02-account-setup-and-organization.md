# Chapter 2: Account Setup & Organization (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Creating a Personal GCP Account (Free Tier)](#part-1-creating-a-personal-gcp-account-free-tier)
- [Part 2: Understanding Projects (GCP's Core Concept)](#part-2-understanding-projects-gcps-core-concept)
- [Part 3: Organization Setup (Company Level)](#part-3-organization-setup-company-level)
- [Part 4: Creating Folders](#part-4-creating-folders)
- [Part 5: Organization Policies (GCP's Guardrails)](#part-5-organization-policies-gcps-guardrails)
- [Part 6: Billing Accounts & Budgets](#part-6-billing-accounts--budgets)

---

## Overview

This chapter covers creating and managing GCP accounts — from a personal free-tier account for learning, to a full enterprise organization setup with folders, projects, and proper governance.

```
What you'll learn:
├── Creating your personal GCP account (free tier + $300 credit)
├── Understanding Projects (GCP's core organizing unit)
├── Organization setup (via Google Workspace / Cloud Identity)
├── Folders & project hierarchy
├── Organization Policies (GCP's guardrails)
├── Cloud Identity (free alternative to Google Workspace)
├── Billing accounts & budgets
├── How companies actually set this up
└── Step-by-step console walkthrough
```

---

## Part 1: Creating a Personal GCP Account (Free Tier)

### What You Need Before Starting

| Requirement | Details |
|-------------|---------|
| Google account | Gmail or Google Workspace account |
| Phone number | For verification |
| Credit/Debit card | Required (won't be charged during free trial) |

### Step-by-Step: Creating an Account

```
Portal: https://cloud.google.com → "Get started for free"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: Sign in with Google Account
┌─────────────────────────────────────────────────────────┐
│ Sign in with your Google account (Gmail)                │
│ or create a new Google account first                    │
│                                                         │
│ 💡 For personal learning: Use your Gmail account        │
│ 💡 For company: Use Google Workspace account            │
└─────────────────────────────────────────────────────────┘

Step 2: Free Trial Information
┌─────────────────────────────────────────────────────────┐
│ Country: [India]                                        │
│                                                         │
│ What describes you best:                                │
│ └── Company size / Role / etc.                         │
│                                                         │
│ Terms of Service: [✓ Accept]                            │
│                                                         │
│ 💡 You get: $300 free credit for 90 days                │
│ 💡 After 90 days or credit used up:                     │
│    - Auto-upgrade to paid (if you choose)              │
│    - Or account paused (nothing deleted for 30 days)   │
│ 💡 You will NOT be charged automatically!               │
│    (Must manually upgrade to paid account)             │
└─────────────────────────────────────────────────────────┘

Step 3: Payment Information
┌─────────────────────────────────────────────────────────┐
│ Account type: ○ Individual  ○ Business                  │
│ Tax information (if India): GST number (optional)       │
│                                                         │
│ Payment method:                                         │
│ Card number:    [XXXX-XXXX-XXXX-XXXX]                  │
│ Expiration:     [MM/YY]                                │
│ CVV:            [XXX]                                   │
│ Cardholder name:[Your Name]                             │
│                                                         │
│ ⚠️ Temporary hold of ~$1 for verification               │
│ 💡 You will NOT be charged during free trial            │
│ 💡 When free trial ends, you must manually upgrade      │
└─────────────────────────────────────────────────────────┘

Step 4: Account Created!
┌─────────────────────────────────────────────────────────┐
│ ✅ Welcome to Google Cloud Platform!                     │
│                                                         │
│ Your free trial:                                        │
│ ├── $300 credit                                        │
│ ├── Valid for 90 days                                  │
│ ├── Access to all GCP services                         │
│ └── Some restrictions (no GPU, limited quotas)         │
│                                                         │
│ A default project "My First Project" is created        │
│ Project ID: something-random-123456                    │
└─────────────────────────────────────────────────────────┘
```

### Immediately After Account Creation (Setup Checklist)

```
⚠️  DAY 1 SETUP TASKS (Do these IMMEDIATELY):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

□ 1. Secure your Google Account
     Google Account → Security → 2-Step Verification → Enable
     Use: Authenticator app or hardware security key
     
□ 2. Create a proper project (don't use "My First Project")
     Console → Project selector → New Project
     - Name: "learning-gcp" or "dev-project"
     - ID: Will be auto-generated (or customize)
     - Organization: "No organization" (for personal)
     
□ 3. Set up Budget Alerts
     Console → Billing → Budgets & alerts → Create budget
     - Budget name: "Monthly Limit"
     - Amount: $10 (or your limit)
     - Alerts at: 50%, 80%, 100%
     - Email notifications: your email
     
□ 4. Enable recommended APIs
     Console → APIs & Services → Enable APIs
     - Compute Engine API
     - Cloud Storage API
     - Cloud Build API
     (APIs are enabled per-project as needed)
     
□ 5. Set up Cloud Shell (familiarize)
     Click the Cloud Shell icon (>_) in top-right
     Run: gcloud config list (verify your setup)
     
□ 6. Review IAM (for personal account)
     Console → IAM & Admin → IAM
     Your Google account should have "Owner" role
```

---

## Part 2: Understanding Projects (GCP's Core Concept)

### What is a Project?

A **Project** is the fundamental organizing entity in GCP. Every single resource you create belongs to exactly one project.

```
┌─────────────────────────────────────────────────────────────┐
│                     GCP PROJECT                              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  A project has three identifiers:                            │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Project Name:   "Production Backend"                    ││
│  │                 (Human-readable, can change, not unique)││
│  │                                                         ││
│  │ Project ID:     "prod-backend-a1b2c3"                  ││
│  │                 (Globally unique, immutable, you choose)││
│  │                 (Used in APIs, CLI, URLs)               ││
│  │                                                         ││
│  │ Project Number: "123456789012"                          ││
│  │                 (Auto-generated, numeric, immutable)    ││
│  │                 (Used internally by GCP)                ││
│  └─────────────────────────────────────────────────────────┘│
│                                                               │
│  What a project contains/controls:                           │
│  ├── All resources (VMs, buckets, databases, etc.)          │
│  ├── Enabled APIs (must enable per project)                 │
│  ├── IAM permissions (who can access what)                  │
│  ├── Billing (linked to a billing account)                  │
│  ├── Quotas (service limits per project)                    │
│  ├── Labels (key-value metadata)                            │
│  └── Audit logs (who did what in this project)              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Creating a Project

```
Console → Project Selector (top bar) → New Project

┌─────────────────────────────────────────────────────────┐
│ Project name:      [My Backend Service]                  │
│                    (Display name, can contain spaces)    │
│                                                         │
│ Project ID:        [my-backend-service-2026]            │
│                    (Auto-generated, click "EDIT" to      │
│                     customize. Must be 6-30 chars,      │
│                     lowercase letters, digits, hyphens) │
│                    ⚠️ CANNOT be changed later!            │
│                    ⚠️ Must be globally unique!            │
│                                                         │
│ Organization:      [No organization] ← personal         │
│                    [company.com] ← if in an org         │
│                                                         │
│ Location:          [No organization] ← personal         │
│                    [folder-name] ← if organizing         │
│                                                         │
│ [CREATE]                                                │
│                                                         │
│ 💡 Naming convention for companies:                      │
│    <company>-<env>-<service>                            │
│    Example: techcorp-prod-backend                       │
│    Example: techcorp-dev-frontend                       │
└─────────────────────────────────────────────────────────┘

CLI equivalent:
  gcloud projects create my-backend-service-2026 \
    --name="My Backend Service" \
    --organization=ORGANIZATION_ID \
    --folder=FOLDER_ID
```

### Project vs AWS Account vs Azure Subscription

```
Comparison:
┌──────────────────────────────────────────────────────────┐
│ GCP Project ≈ AWS Account ≈ Azure Subscription           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Similarities:                                            │
│ ├── Isolation boundary for resources                    │
│ ├── Billing boundary                                    │
│ ├── IAM boundary                                        │
│ └── Has its own quotas/limits                           │
│                                                          │
│ Key Differences:                                         │
│ ├── GCP: One person can create 100s of projects easily  │
│ │   AWS: Each account needs unique email, more overhead │
│ │                                                        │
│ ├── GCP: Projects are lightweight (just a logical unit)  │
│ │   AWS: Accounts are heavyweight (separate root user)  │
│ │                                                        │
│ ├── GCP: Billing account shared across projects         │
│ │   AWS: Each account has own payment (consolidated)    │
│ │                                                        │
│ └── GCP: APIs must be enabled per project               │
│     AWS: Services available by default in account       │
└──────────────────────────────────────────────────────────┘
```

---

## Part 3: Organization Setup (Company Level)

### Two Paths to GCP Organization

```
┌─────────────────────────────────────────────────────────────┐
│            HOW TO GET A GCP ORGANIZATION                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Path 1: Google Workspace (Paid)                             │
│  ─────────────────────────────────────────                   │
│  If your company uses Google Workspace (Gmail, Drive, etc.) │
│  → GCP Organization is auto-created for your domain         │
│  → Users are your Google Workspace users                    │
│  → Admin: admin.google.com → control everything             │
│                                                               │
│  Path 2: Cloud Identity Free (Recommended for Azure AD/Okta)│
│  ─────────────────────────────────────────────                │
│  If your company does NOT use Google Workspace               │
│  → Sign up for Cloud Identity Free                          │
│  → Get an Organization node for your domain                 │
│  → Federate with your existing IdP (Okta, Azure AD)         │
│  → No Google Workspace licensing needed                     │
│                                                               │
│  Path 3: No Organization (Personal / Small projects)         │
│  ────────────────────────────────────────────                 │
│  Just use a Gmail account                                    │
│  → Projects have no organization node                       │
│  → No folders, no org policies                              │
│  → Fine for learning, NOT for production                    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Setting Up Cloud Identity Free (for Organization)

```
Step-by-Step: Getting an Organization without Google Workspace:

1. Go to: https://cloud.google.com/identity/docs/set-up-cloud-identity-admin
   or: https://workspace.google.com/signup/gcpidentity/welcome

2. Sign up with your company domain:
┌─────────────────────────────────────────────────────────┐
│ Business name:    [TechCorp Inc.]                       │
│ Employees:        [Choose range]                        │
│ Country:          [India]                               │
│                                                         │
│ Domain:           [techcorp.com]                         │
│ (You must own this domain and verify it)               │
│                                                         │
│ Admin email:      [admin@techcorp.com]                  │
│ Admin password:   [Strong password]                     │
└─────────────────────────────────────────────────────────┘

3. Verify domain ownership:
   - Add TXT record to your DNS:
     google-site-verification=XXXXXXXXXXXXXXX
   - Or upload HTML file to your website

4. Once verified:
   - Organization node created: techcorp.com
   - Super Admin: admin@techcorp.com
   - Now you can create folders, apply org policies
   - Users from your domain auto-belong to this org

5. (Optional) Federate with external IdP:
   Admin Console → Security → SSO with third-party IdP
   → Configure SAML with Okta/Azure AD/your IdP
```

### Organization Resource Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│               GCP RESOURCE HIERARCHY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Organization (techcorp.com)     ← Top level, tied to domain    │
│  │                                                               │
│  ├── Folder: Production          ← Group projects logically     │
│  │   ├── Project: prod-frontend  ← Contains actual resources    │
│  │   │   ├── Compute Engine VMs                                 │
│  │   │   ├── Cloud Run services                                 │
│  │   │   └── Cloud Storage buckets                              │
│  │   ├── Project: prod-backend                                  │
│  │   │   ├── GKE clusters                                       │
│  │   │   └── Cloud SQL instances                                │
│  │   └── Project: prod-data                                     │
│  │       ├── BigQuery datasets                                  │
│  │       └── Pub/Sub topics                                     │
│  │                                                               │
│  ├── Folder: Non-Production                                     │
│  │   ├── Folder: Staging         ← Folders can be nested!      │
│  │   │   └── Project: staging-app                               │
│  │   └── Folder: Development                                    │
│  │       └── Project: dev-app                                   │
│  │                                                               │
│  ├── Folder: Shared Services                                    │
│  │   ├── Project: shared-networking (Shared VPC Host)           │
│  │   ├── Project: shared-cicd                                   │
│  │   └── Project: shared-monitoring                             │
│  │                                                               │
│  └── Folder: Sandbox                                            │
│      └── Project: sandbox-experiments                           │
│                                                                   │
│  KEY RULE: IAM policies INHERIT downward                        │
│  Organization → Folder → Project → Resource                    │
│  (More permissive as you go up, more restrictive as you go down)│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Inheritance Diagram

```
IAM Policy Inheritance:

  Organization: techcorp.com
  IAM: roles/viewer → user:auditor@techcorp.com
  (Auditor can VIEW everything in the entire organization)
       │
       ▼ INHERITS
  Folder: Production
  IAM: roles/editor → group:devops@techcorp.com
  (DevOps can EDIT everything in Production folder)
       │
       ▼ INHERITS
  Project: prod-backend
  IAM: roles/cloudsql.admin → user:dba@techcorp.com
  (DBA can manage Cloud SQL in this specific project)
       │
       ▼ INHERITS
  Resource: Cloud SQL Instance "prod-db"
  (All above policies apply to this resource)

  Effective permissions on "prod-db":
  ├── auditor@techcorp.com → Viewer (from Org)
  ├── devops@techcorp.com → Editor (from Folder)
  └── dba@techcorp.com → Cloud SQL Admin (from Project)
  
  ⚠️ You CANNOT remove inherited permissions at a lower level!
     You can only ADD more permissions or use deny policies.
```

---

## Part 4: Creating Folders

```
Console → IAM & Admin → Manage Resources → Create Folder
(or from Organization view)

┌─────────────────────────────────────────────────────────┐
│ Folder name:     [Production]                           │
│ Parent:          [Organization: techcorp.com]           │
│                  or [Folder: parent-folder]             │
│                                                         │
│ [CREATE]                                                │
│                                                         │
│ 💡 Folders can be nested up to 10 levels deep           │
│ 💡 Use for: environments, teams, business units         │
└─────────────────────────────────────────────────────────┘

CLI:
  gcloud resource-manager folders create \
    --display-name="Production" \
    --organization=ORGANIZATION_ID
    
  # Or nested under another folder:
  gcloud resource-manager folders create \
    --display-name="Staging" \
    --folder=PARENT_FOLDER_ID
```

### Recommended Folder Structures

```
Structure 1: By Environment (Most Common)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Organization
├── Production/
│   ├── project-prod-app
│   └── project-prod-data
├── Non-Production/
│   ├── Staging/
│   │   └── project-staging
│   └── Development/
│       └── project-dev
├── Shared/
│   ├── project-shared-networking
│   └── project-shared-cicd
└── Sandbox/
    └── project-sandbox

Structure 2: By Team/Business Unit
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Organization
├── Team-Frontend/
│   ├── frontend-prod
│   ├── frontend-staging
│   └── frontend-dev
├── Team-Backend/
│   ├── backend-prod
│   └── backend-dev
├── Team-Data/
│   ├── data-prod
│   └── data-dev
└── Shared/
    └── shared-infra

Structure 3: Hybrid (Environment + Team)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Organization
├── Production/
│   ├── prod-team-a
│   └── prod-team-b
├── Non-Production/
│   ├── dev-team-a
│   └── dev-team-b
└── Infrastructure/
    ├── shared-networking
    └── shared-security
```

---

## Part 5: Organization Policies (GCP's Guardrails)

### What are Organization Policies?

Organization Policies are constraints applied at the Organization, Folder, or Project level that restrict what users can do — regardless of their IAM permissions.

```
Organization Policies vs IAM:

  IAM: "WHO can do WHAT" (Grant permissions)
  Org Policy: "WHAT is ALLOWED to be done" (Restrict actions)

  Example:
  - IAM says: User has roles/compute.admin (can create VMs anywhere)
  - Org Policy says: VMs can only be created in asia-south1 and us-central1
  - Result: User can create VMs, but ONLY in those two regions

  Think of it as:
  ┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
  │  IAM Policy │  ∩  │  Org Policy     │  =  │  What can   │
  │  (Grants)   │     │  (Constraints)  │     │  actually   │
  │             │     │                 │     │  be done    │
  └─────────────┘     └─────────────────┘     └─────────────┘
```

### Setting Organization Policies

```
Console → IAM & Admin → Organization Policies
(Must be at Organization or Folder level)

Common Organization Policies:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 1. Restrict Resource Locations (Most Important!)

```
Console → Organization Policies → "Resource Location Restriction"
(constraints/gcp.resourceLocations)

┌─────────────────────────────────────────────────────────┐
│ Policy: Resource Location Restriction                   │
│                                                         │
│ Applies to: [Organization / Folder / Project]           │
│                                                         │
│ Policy enforcement: [Replace]                           │
│                                                         │
│ Allowed values:                                         │
│ ├── asia-south1 (Mumbai)                               │
│ ├── asia-south2 (Delhi)                                │
│ ├── us-central1 (Iowa)                                 │
│ └── in (all India locations)                           │
│                                                         │
│ Result: Resources can ONLY be created in these         │
│ locations. Any other region → blocked!                  │
│                                                         │
│ 💡 Use "in:" prefix for countries: in:us, in:eu, in:asia│
└─────────────────────────────────────────────────────────┘
```

#### 2. Disable Default Service Account Creation

```
(constraints/iam.automaticIamGrantsForDefaultServiceAccounts)

┌─────────────────────────────────────────────────────────┐
│ Policy: Disable Automatic IAM Grants for Default SA     │
│                                                         │
│ Enforcement: [Enforce]                                  │
│                                                         │
│ What it does:                                           │
│ By default, GCP auto-creates a service account with    │
│ Editor role when you enable Compute Engine API.         │
│ This policy STOPS that (security best practice!)       │
│                                                         │
│ ⚠️ ALWAYS enable this in production!                    │
└─────────────────────────────────────────────────────────┘
```

#### 3. Restrict VM External IPs

```
(constraints/compute.vmExternalIpAccess)

┌─────────────────────────────────────────────────────────┐
│ Policy: Define allowed external IPs for VM instances    │
│                                                         │
│ Enforcement: [Deny All]                                 │
│                                                         │
│ What it does:                                           │
│ Prevents VMs from having public/external IP addresses  │
│ Forces all traffic through Cloud NAT or Load Balancer  │
│                                                         │
│ 💡 Apply to Production folder                           │
│ 💡 Exempt specific VMs if needed (e.g., bastion host)  │
└─────────────────────────────────────────────────────────┘
```

#### 4. Require OS Login

```
(constraints/compute.requireOsLogin)

┌─────────────────────────────────────────────────────────┐
│ Policy: Require OS Login for SSH                        │
│                                                         │
│ Enforcement: [Enforce]                                  │
│                                                         │
│ What it does:                                           │
│ Forces all SSH access to VMs to go through OS Login    │
│ (uses IAM for SSH instead of SSH keys in metadata)    │
│ Better auditing and access control                    │
└─────────────────────────────────────────────────────────┘
```

#### 5. Restrict Shared VPC Subnetworks

```
(constraints/compute.restrictSharedVpcSubnetworks)

┌─────────────────────────────────────────────────────────┐
│ Policy: Restrict which Shared VPC subnets can be used  │
│                                                         │
│ What it does:                                           │
│ Controls which subnets from a Shared VPC host project  │
│ can be used by service projects                        │
│ Prevents teams from using wrong subnets               │
└─────────────────────────────────────────────────────────┘
```

### Full List of Common Org Policies

```
┌──────────────────────────────────────────────────────────────────────┐
│ Constraint                          │ Purpose                         │
├──────────────────────────────────────────────────────────────────────┤
│ gcp.resourceLocations              │ Restrict which regions allowed  │
│ compute.vmExternalIpAccess         │ Block public IPs on VMs         │
│ compute.requireOsLogin             │ Force OS Login for SSH          │
│ compute.disableSerialPortAccess    │ Block serial port (security)    │
│ iam.disableServiceAccountKeyCreation│ Block SA key downloads         │
│ iam.automaticIamGrantsForDefault...│ Block auto Editor grants        │
│ sql.restrictPublicIp               │ Block public IPs on Cloud SQL   │
│ storage.uniformBucketLevelAccess   │ Force uniform access on buckets │
│ compute.restrictVpcPeering         │ Control VPC peering             │
│ compute.skipDefaultNetworkCreation │ Don't create default VPC        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Billing Accounts & Budgets

### Billing Structure

```
┌─────────────────────────────────────────────────────────────┐
│                   GCP BILLING STRUCTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Billing Account (Company Credit Card / Invoice)             │
│  ├── Linked to: Project A → charges go to this billing acct │
│  ├── Linked to: Project B → charges go to this billing acct │
│  ├── Linked to: Project C → charges go to this billing acct │
│  └── Linked to: Project D → charges go to this billing acct │
│                                                               │
│  You can have MULTIPLE billing accounts:                     │
│  ├── Billing Account: "Development" (department card)        │
│  │   ├── Project: dev-frontend                              │
│  │   └── Project: dev-backend                               │
│  └── Billing Account: "Production" (company account)         │
│      ├── Project: prod-app                                  │
│      └── Project: prod-data                                 │
│                                                               │
│  Types of billing accounts:                                  │
│  ├── Self-Serve: Credit card, pay as you go                 │
│  ├── Invoiced: Monthly invoice (for large customers)         │
│  └── Reseller: Through Google Cloud Partner                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Creating a Budget

```
Console → Billing → Budgets & alerts → Create budget

┌─────────────────────────────────────────────────────────┐
│                    CREATE BUDGET                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Step 1: Scope                                           │
│ ┌─────────────────────────────────────────────────────┐│
│ │ Name: [Monthly Production Budget]                   ││
│ │                                                     ││
│ │ Time range: [Monthly / Quarterly / Yearly / Custom] ││
│ │                                                     ││
│ │ Projects:                                           ││
│ │ ○ All projects in billing account                   ││
│ │ ○ Select specific projects: [prod-backend ✓]       ││
│ │                                                     ││
│ │ Services:                                           ││
│ │ ○ All services                                      ││
│ │ ○ Select: [Compute Engine ✓] [Cloud SQL ✓]         ││
│ │                                                     ││
│ │ Labels: (filter by label)                           ││
│ │ [env: production]                                   ││
│ └─────────────────────────────────────────────────────┘│
│                                                         │
│ Step 2: Amount                                          │
│ ┌─────────────────────────────────────────────────────┐│
│ │ Budget type:                                        ││
│ │ ○ Specified amount: [$1,000]                        ││
│ │ ○ Last month's spend                               ││
│ │ ○ Last period's spend                              ││
│ │                                                     ││
│ │ Include credits: [✓] (subtract free tier credits)   ││
│ └─────────────────────────────────────────────────────┘│
│                                                         │
│ Step 3: Actions (Alerts)                                │
│ ┌─────────────────────────────────────────────────────┐│
│ │ Alert thresholds:                                   ││
│ │ ├── 50% of budget → Email alert                    ││
│ │ ├── 80% of budget → Email alert                    ││
│ │ ├── 100% of budget → Email alert                   ││
│ │ └── 120% of budget → Email alert (forecasted)     ││
│ │                                                     ││
│ │ Notifications:                                      ││
│ │ ├── [✓] Email to billing admins                    ││
│ │ ├── [✓] Email to custom recipients                 ││
│ │ │       [devops@company.com]                       ││
│ │ ├── [ ] Connect Pub/Sub topic (for automation)     ││
│ │ │       (Can trigger Cloud Function to stop VMs!)  ││
│ │ └── [ ] Connect monitoring notification channel    ││
│ └─────────────────────────────────────────────────────┘│
│                                                         │
│ [CREATE]                                                │
│                                                         │
│ ⚠️ Budgets DON'T stop spending! They only alert you.    │
│ 💡 To auto-stop: Use Pub/Sub + Cloud Function to        │
│    disable billing or stop resources when budget hit.  │
└─────────────────────────────────────────────────────────┘
```

### Billing IAM Roles

```
Who can do what with billing:

┌──────────────────────────────────────────────────────────────────┐
│ Role                        │ What they can do                    │
├──────────────────────────────────────────────────────────────────┤
│ Billing Account Admin       │ Manage billing account, link/unlink│
│                             │ projects, manage payments           │
├──────────────────────────────────────────────────────────────────┤
│ Billing Account User        │ Link projects to billing account   │
│                             │ (needed to create projects)        │
├──────────────────────────────────────────────────────────────────┤
│ Billing Account Viewer      │ View billing info, transactions    │
├──────────────────────────────────────────────────────────────────┤
│ Project Billing Manager     │ Link/unlink specific project       │
└──────────────────────────────────────────────────────────────────┘

💡 Separation of duties:
   - Developers: NO billing access (can't see costs)
   - Finance team: Billing Viewer (see costs, can't change)
   - Cloud admin: Billing Admin (full control)
   - Project lead: Billing User (can create projects with billing)
```

---

## Part 7: Shared VPC (Centralized Networking)

### What is Shared VPC?

```
┌─────────────────────────────────────────────────────────────────┐
│                      SHARED VPC CONCEPT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Without Shared VPC:                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ Project A│  │ Project B│  │ Project C│                     │
│  │ Own VPC  │  │ Own VPC  │  │ Own VPC  │                     │
│  │ 10.0.0/16│  │ 10.1.0/16│  │ 10.2.0/16│                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
│  Problem: Each team manages own network, no central control    │
│  Problem: Need VPC Peering between all (N² connections)        │
│                                                                   │
│  With Shared VPC:                                                │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  HOST PROJECT (shared-networking)                     │      │
│  │  VPC: company-vpc (10.0.0.0/8)                       │      │
│  │  ├── Subnet: prod (10.0.0.0/16)                     │      │
│  │  ├── Subnet: staging (10.1.0.0/16)                  │      │
│  │  └── Subnet: dev (10.2.0.0/16)                      │      │
│  │  Firewall rules managed centrally                    │      │
│  │  Cloud Router, Cloud NAT managed centrally           │      │
│  └──────────────────────┬───────────────────────────────┘      │
│                          │ Shared to:                            │
│         ┌────────────────┼────────────────┐                     │
│         ▼                ▼                ▼                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ Service  │  │ Service  │  │ Service  │                     │
│  │ Project A│  │ Project B│  │ Project C│                     │
│  │ Uses     │  │ Uses     │  │ Uses     │                     │
│  │ prod     │  │ staging  │  │ dev      │                     │
│  │ subnet   │  │ subnet   │  │ subnet   │                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
│  Benefit: Central network team controls networking             │
│  Benefit: Service teams just deploy resources                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Real-World Company Scenarios

### Scenario 1: Startup (5-10 developers)

```
Organization: startup.com (Cloud Identity Free)
│
├── Folder: Production
│   └── Project: startup-prod
│       ├── Cloud Run (web app)
│       ├── Cloud SQL (database)
│       └── Cloud Storage (assets)
│
├── Folder: Non-Production
│   └── Project: startup-dev
│
└── Folder: Sandbox
    └── Project: startup-sandbox

Billing: Single billing account (company card)
IAM: Google Groups for team permissions
Org Policies: Restrict to 1-2 regions
```

### Scenario 2: Mid-Size Company (50-100 developers)

```
Organization: techcorp.com (Google Workspace or Cloud Identity)
│
├── Folder: Production
│   ├── Project: techcorp-prod-frontend
│   ├── Project: techcorp-prod-backend
│   ├── Project: techcorp-prod-data
│   └── Project: techcorp-prod-ml
│
├── Folder: Shared Services
│   ├── Project: techcorp-host-networking (Shared VPC Host)
│   ├── Project: techcorp-cicd
│   ├── Project: techcorp-monitoring
│   └── Project: techcorp-security
│
├── Folder: Non-Production
│   ├── Folder: Staging
│   │   └── Project: techcorp-staging
│   └── Folder: Development
│       └── Project: techcorp-dev
│
└── Folder: Sandbox
    ├── Project: techcorp-sandbox-team-a
    └── Project: techcorp-sandbox-team-b

Billing:
├── Billing Account: "Production" (invoiced)
├── Billing Account: "Non-Production" (credit card)
└── Budget alerts per project

Org Policies:
├── Resource locations: asia-south1, us-central1 only
├── No external IPs on VMs in Production
├── No SA key creation
├── Uniform bucket access enforced
└── Skip default network creation

IAM:
├── Google Groups synced from IdP
├── Service accounts per project (no key downloads)
├── Workload Identity for GKE
└── Audit logging enabled
```

### Scenario 3: Enterprise (500+ developers)

```
Organization: enterprise.com (Cloud Identity + Okta Federation)
│
├── Folder: Platform
│   ├── Project: platform-networking-host (Shared VPC)
│   ├── Project: platform-dns
│   ├── Project: platform-logging (Log sinks from all)
│   ├── Project: platform-monitoring (Metrics scopes)
│   ├── Project: platform-security (Security Command Center)
│   └── Project: platform-cicd (Cloud Build + Artifact Registry)
│
├── Folder: Business Unit A
│   ├── Folder: BU-A Production
│   │   ├── Project: bua-prod-api
│   │   ├── Project: bua-prod-worker
│   │   └── Project: bua-prod-data
│   └── Folder: BU-A Non-Production
│       ├── Project: bua-staging
│       └── Project: bua-dev
│
├── Folder: Business Unit B
│   ├── Folder: BU-B Production
│   └── Folder: BU-B Non-Production
│
├── Folder: Data Platform
│   ├── Project: data-lake-prod
│   ├── Project: data-analytics
│   └── Project: data-ml
│
└── Folder: Sandbox (auto-cleanup after 30 days)
    └── Project: sandbox-*

Org Policies (by folder):
├── Root level: Restrict regions, no default SA grants
├── Production: No public IPs, no SA keys, enforce CMEK
├── Platform: Restricted VPC Service Controls
└── Sandbox: Allowed to experiment (relaxed policies)

VPC Service Controls:
├── Perimeter around Production projects
├── Prevents data exfiltration
└── Access levels for specific users/IPs

Billing:
├── Sub-accounts per business unit
├── Committed Use Discounts at org level
├── BigQuery flat-rate reservations shared
└── Monthly cost review per BU
```

---

## Part 9: Account Security Best Practices Summary

```
┌─────────────────────────────────────────────────────────────────┐
│              GCP ACCOUNT SECURITY BEST PRACTICES                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ORGANIZATION LEVEL:                                             │
│  ├── ✅ Enable 2-Step Verification for all users                │
│  ├── ✅ Set up Organization Policies (regions, IPs, SA keys)    │
│  ├── ✅ Skip default network creation (org policy)              │
│  ├── ✅ Disable automatic IAM grants for default SAs            │
│  ├── ✅ Enable Security Command Center (at least free tier)     │
│  └── ✅ Configure audit log retention                           │
│                                                                   │
│  PROJECT LEVEL:                                                  │
│  ├── ✅ Use dedicated service accounts (not default)            │
│  ├── ✅ Never download SA keys (use Workload Identity instead)  │
│  ├── ✅ Enable only needed APIs                                 │
│  ├── ✅ Set up budget alerts                                    │
│  └── ✅ Apply labels for cost tracking                          │
│                                                                   │
│  IAM:                                                            │
│  ├── ✅ Use Google Groups for permissions (not individual users)│
│  ├── ✅ Principle of least privilege                             │
│  ├── ✅ Use predefined roles over primitive (Owner/Editor/Viewer)│
│  ├── ✅ Regular IAM audit (IAM Recommender)                     │
│  ├── ✅ Federate with corporate IdP                             │
│  └── ✅ Use IAM Conditions for time-based/resource-based access │
│                                                                   │
│  NETWORKING:                                                     │
│  ├── ✅ Use Shared VPC for centralized network control          │
│  ├── ✅ Private Google Access enabled                           │
│  ├── ✅ No public IPs on VMs (use Cloud NAT + IAP for access)  │
│  ├── ✅ VPC Flow Logs enabled                                   │
│  └── ✅ VPC Service Controls for sensitive data                 │
│                                                                   │
│  MONITORING:                                                     │
│  ├── ✅ Cloud Audit Logs (Admin Activity = always on)           │
│  ├── ✅ Data Access logs enabled for sensitive services         │
│  ├── ✅ Export logs to separate project (for retention)         │
│  ├── ✅ Alert on suspicious IAM changes                        │
│  └── ✅ Security Command Center for vulnerability scanning     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: Console Navigation

| Task | Console Path |
|------|-------------|
| Create Project | Top bar → Project Selector → New Project |
| View Organization | IAM & Admin → Manage Resources |
| Create Folder | IAM & Admin → Manage Resources → Create Folder |
| Organization Policies | IAM & Admin → Organization Policies |
| Billing | Navigation Menu → Billing |
| Create Budget | Billing → Budgets & alerts → Create budget |
| Link Project to Billing | Billing → Account management → Link project |
| Manage IAM | IAM & Admin → IAM |
| Enable APIs | APIs & Services → Enable APIs & Services |
| View Audit Logs | Logging → Logs Explorer → filter: protoPayload.@type="type.googleapis.com/google.cloud.audit.v1.AuditLog" |

---

## What's Next?

In the next chapter, we'll dive deep into the resource hierarchy — how resources are organized within projects, tagging/labels strategy, and how to manage resources at scale.

→ Next: [Chapter 3: Resource Hierarchy & Projects](03-resource-hierarchy.md)

---

*Last Updated: May 2026*
