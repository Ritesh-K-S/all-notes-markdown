# Chapter 4: Microsoft Entra ID & Azure RBAC (Azure)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Microsoft Entra ID Fundamentals](#part-1-microsoft-entra-id-fundamentals)
- [Part 2: Users](#part-2-users)
- [Part 3: Groups](#part-3-groups)
- [Part 4: App Registrations & Service Principals](#part-4-app-registrations--service-principals)
- [Part 5: Managed Identities](#part-5-managed-identities)
- [Part 6: Azure RBAC (Role-Based Access Control)](#part-6-azure-rbac-role-based-access-control)
- [Part 7: Conditional Access](#part-7-conditional-access)
- [Part 8: Privileged Identity Management (PIM)](#part-8-privileged-identity-management-pim)
- [Part 9: Multi-Factor Authentication (MFA)](#part-9-multi-factor-authentication-mfa)
- [Part 10: Entra ID Roles vs Azure RBAC Roles](#part-10-entra-id-roles-vs-azure-rbac-roles)
- [Part 11: Security Best Practices](#part-11-security-best-practices)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference: CLI Commands](#quick-reference-cli-commands)
- [What's Next?](#whats-next)

---

## Overview

Azure identity and access management has two pillars: **Microsoft Entra ID** (formerly Azure AD) handles identity/authentication, and **Azure RBAC** handles authorization to Azure resources. This chapter covers both comprehensively.

```
What you'll learn:
├── Microsoft Entra ID fundamentals (tenant, users, groups)
├── User management (creation, types, properties)
├── Group management (types, dynamic groups, nesting)
├── App Registrations & Service Principals
├── Managed Identities (system-assigned vs user-assigned)
├── Azure RBAC (roles, scopes, assignments)
├── Built-in Roles vs Custom Roles
├── Conditional Access (zero-trust policies)
├── Privileged Identity Management (PIM)
├── Multi-Factor Authentication (MFA)
├── External Identities (B2B, B2C)
├── Security best practices
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: Microsoft Entra ID Fundamentals

### What is Microsoft Entra ID?

```
┌─────────────────────────────────────────────────────────────────────┐
│              MICROSOFT ENTRA ID (formerly Azure AD)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Cloud-based identity and access management service             │
│ Purpose: Authentication + Authorization + Identity Governance        │
│                                                                       │
│ Key Concepts:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ TENANT:                                                       │    │
│ │ ├── A dedicated Entra ID instance for your organization      │    │
│ │ ├── Created automatically with Azure subscription            │    │
│ │ ├── Identified by: tenantname.onmicrosoft.com                │    │
│ │ ├── Custom domain: techcorp.com (verified)                   │    │
│ │ ├── Tenant ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (GUID)  │    │
│ │ └── ALL Azure subscriptions trust exactly ONE tenant         │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Entra ID vs On-Premises AD:                                          │
│ ┌──────────────────────┬──────────────────────────────────────┐     │
│ │ On-Premises AD       │ Microsoft Entra ID                    │     │
│ ├──────────────────────┼──────────────────────────────────────┤     │
│ │ LDAP                 │ REST APIs (Microsoft Graph)           │     │
│ │ Kerberos/NTLM        │ SAML / OAuth 2.0 / OpenID Connect   │     │
│ │ OUs (Org Units)      │ Flat structure (no OUs)              │     │
│ │ Group Policy (GPO)   │ Conditional Access + Intune          │     │
│ │ Domain Controllers   │ Cloud service (no servers to manage) │     │
│ │ Forest/Domain        │ Tenant                                │     │
│ │ Computer objects     │ Devices (registered/joined)          │     │
│ └──────────────────────┴──────────────────────────────────────┘     │
│                                                                       │
│ Entra ID Editions:                                                   │
│ ├── Free: Basic user/group, SSO (10 apps), MFA               │
│ ├── P1: Conditional Access, dynamic groups, self-service reset│
│ ├── P2: PIM, Identity Protection, access reviews             │
│ └── Governance: Lifecycle workflows, entitlement management   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Users

### Types of Users

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER TYPES                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 1. MEMBER USERS (internal)                                       │
│    ├── Full members of your tenant                              │
│    ├── john.doe@techcorp.com                                    │
│    ├── Created directly or synced from on-premises AD           │
│    └── Full access to tenant features                           │
│                                                                   │
│ 2. GUEST USERS (external — B2B)                                 │
│    ├── External collaborators invited to your tenant            │
│    ├── contractor@external.com (invited as guest)               │
│    ├── Limited default permissions                              │
│    ├── Can be given Azure RBAC roles                            │
│    └── Identity managed by their home organization              │
│                                                                   │
│ 3. SYNCED USERS (hybrid)                                        │
│    ├── Synced from on-premises AD via Entra Connect             │
│    ├── Source of authority: on-premises AD                      │
│    ├── Changes made on-prem, synced to cloud                    │
│    └── Enables hybrid identity (same creds everywhere)          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a User

```
Portal → Microsoft Entra ID → Users → New user → Create new user

┌─────────────────────────────────────────────────────────────────┐
│                   CREATE NEW USER                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Basics tab:                                                      │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ User principal name: [john.doe] @ [techcorp.com ▼]        │   │
│ │   (This is the sign-in name: john.doe@techcorp.com)       │   │
│ │                                                            │   │
│ │ Mail nickname:       [john.doe] (auto from UPN)           │   │
│ │ Display name:        [John Doe]                            │   │
│ │ Password:            [Auto-generate ▼] or set manually    │   │
│ │   ☑ Require change at first login                         │   │
│ │ Account enabled:     ☑ Yes                                │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Properties tab:                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Identity:                                                  │   │
│ │   First name:        [John]                                │   │
│ │   Last name:         [Doe]                                 │   │
│ │   User type:         [Member ▼]  (Member or Guest)        │   │
│ │                                                            │   │
│ │ Job information:                                           │   │
│ │   Job title:         [Senior Developer]                    │   │
│ │   Department:        [Engineering]                         │   │
│ │   Company name:      [TechCorp]                            │   │
│ │   Manager:           [jane.smith@techcorp.com]             │   │
│ │                                                            │   │
│ │ Contact info:                                              │   │
│ │   Email:             [john.doe@techcorp.com]               │   │
│ │   Mobile phone:      [+91-9876543210]                      │   │
│ │   Office location:   [Mumbai, India]                       │   │
│ │                                                            │   │
│ │ Settings:                                                  │   │
│ │   Usage location:    [India ▼] (required for licensing!)   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Assignments tab:                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Add group:   [sg-engineering ▼]                            │   │
│ │ Add role:    [None ▼]  (Entra ID directory roles)         │   │
│ │              (Global Admin, User Admin, etc.)              │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Review + create]                                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  az ad user create \
    --display-name "John Doe" \
    --user-principal-name john.doe@techcorp.com \
    --password "TempP@ss123!" \
    --force-change-password-next-sign-in true \
    --department "Engineering" \
    --job-title "Senior Developer"

PowerShell:
  New-MgUser -DisplayName "John Doe" `
    -UserPrincipalName "john.doe@techcorp.com" `
    -PasswordProfile @{Password="TempP@ss123!"; ForceChangePasswordNextSignIn=$true} `
    -AccountEnabled $true `
    -Department "Engineering" `
    -JobTitle "Senior Developer" `
    -UsageLocation "IN"
```

---

## Part 3: Groups

### Types of Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GROUP TYPES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. SECURITY GROUPS                                                   │
│    ├── Purpose: Manage access to Azure resources                    │
│    ├── Can be used in: RBAC assignments, Conditional Access         │
│    ├── Members: Users, other groups, service principals, devices    │
│    └── Most common for Azure resource access management             │
│                                                                       │
│ 2. MICROSOFT 365 GROUPS                                              │
│    ├── Purpose: Collaboration (shared mailbox, Teams, SharePoint)   │
│    ├── Not typically used for Azure RBAC                            │
│    ├── Members: Users only (no nesting, no service principals)      │
│    └── Auto-creates shared resources (mailbox, site, etc.)          │
│                                                                       │
│ MEMBERSHIP TYPES:                                                    │
│ ┌─────────────────────────────────────────────────────────────┐     │
│ │ Assigned:                                                    │     │
│ │ ├── Manually add/remove members                             │     │
│ │ ├── Admin explicitly controls membership                    │     │
│ │ └── Good for: Small groups, special access                  │     │
│ │                                                              │     │
│ │ Dynamic User: (requires P1 license)                         │     │
│ │ ├── Membership based on user attribute rules                │     │
│ │ ├── Auto-add users matching criteria                        │     │
│ │ ├── Auto-remove when no longer matching                     │     │
│ │ ├── Rule example: user.department -eq "Engineering"         │     │
│ │ └── Good for: Automatic team-based access                   │     │
│ │                                                              │     │
│ │ Dynamic Device: (requires P1 license)                       │     │
│ │ ├── Membership based on device attributes                   │     │
│ │ ├── Rule example: device.deviceOSType -eq "Windows"         │     │
│ │ └── Good for: Device-based Conditional Access               │     │
│ └─────────────────────────────────────────────────────────────┘     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Groups

```
Portal → Microsoft Entra ID → Groups → New group

┌─────────────────────────────────────────────────────────────────┐
│                    CREATE GROUP                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Group type:         [Security ▼]                                 │
│ Group name:         [sg-backend-developers]                      │
│ Group description:  [Backend development team members]            │
│ Entra ID roles:     ☐ can be assigned (P1, for directory roles) │
│                                                                   │
│ Membership type:    [Dynamic User ▼]                             │
│                                                                   │
│ Dynamic query:                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Rule builder:                                              │   │
│ │ Property: [department]                                     │   │
│ │ Operator: [Equals]                                        │   │
│ │ Value:    [Engineering]                                    │   │
│ │                                                            │   │
│ │ AND                                                        │   │
│ │                                                            │   │
│ │ Property: [jobTitle]                                       │   │
│ │ Operator: [Contains]                                      │   │
│ │ Value:    [Backend]                                        │   │
│ │                                                            │   │
│ │ Rule syntax:                                               │   │
│ │ (user.department -eq "Engineering") and                    │   │
│ │ (user.jobTitle -contains "Backend")                        │   │
│ │                                                            │   │
│ │ [Validate rules] ← Test with specific user                │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Owners:            [+ Add owners] (who manages the group)        │
│ Members:           (auto-populated by dynamic rule)              │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create security group (assigned membership)
  az ad group create \
    --display-name "sg-backend-developers" \
    --mail-nickname "sg-backend-developers" \
    --description "Backend development team"

  # Add member
  az ad group member add \
    --group "sg-backend-developers" \
    --member-id USER_OBJECT_ID
```

### Recommended Group Structure

```
Naming convention: <type>-<purpose>-<scope>
  sg = security group

┌─────────────────────────────────────────────────────────────────┐
│              RECOMMENDED GROUPS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ By Team (dynamic membership):                                    │
│ ├── sg-team-backend      (department=Engineering + title Backend)│
│ ├── sg-team-frontend     (department=Engineering + title Frontend│
│ ├── sg-team-data         (department=Data Engineering)          │
│ ├── sg-team-devops       (department=DevOps/Platform)           │
│ └── sg-team-security     (department=Security)                  │
│                                                                   │
│ By Azure Role (assigned membership):                             │
│ ├── sg-role-subscription-contributor (sub-level Contributor)    │
│ ├── sg-role-prod-reader              (prod read-only)           │
│ ├── sg-role-sql-admin                (SQL DB access)            │
│ ├── sg-role-keyvault-reader          (secrets access)           │
│ └── sg-role-aks-cluster-admin        (AKS admin)               │
│                                                                   │
│ By Environment (for Conditional Access):                         │
│ ├── sg-env-prod-access   (users allowed to access prod)         │
│ ├── sg-env-staging-access                                       │
│ └── sg-env-dev-access                                           │
│                                                                   │
│ Special:                                                         │
│ ├── sg-emergency-access  (break-glass accounts)                 │
│ ├── sg-mfa-exclusion     (for service accounts/rooms)           │
│ └── sg-external-vendors  (guest users from vendors)             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: App Registrations & Service Principals

### What Are They?

```
┌─────────────────────────────────────────────────────────────────────┐
│          APP REGISTRATION vs SERVICE PRINCIPAL                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ APP REGISTRATION (the application definition):                       │
│ ├── A "blueprint" for your application in Entra ID                  │
│ ├── Defines: client ID, redirect URIs, permissions needed           │
│ ├── Lives in your HOME tenant                                       │
│ ├── Think of it as the "class" in OOP                               │
│ └── Created once per app                                            │
│                                                                       │
│ SERVICE PRINCIPAL (the instance in a tenant):                        │
│ ├── The "instance" of the app registration in a specific tenant     │
│ ├── What actually gets permissions (RBAC roles)                     │
│ ├── Created automatically when app is consented to                  │
│ ├── Think of it as the "object" instantiated from the class         │
│ └── Each tenant using the app has its own service principal         │
│                                                                       │
│ Analogy:                                                             │
│ App Registration = Recipe (definition)                               │
│ Service Principal = The actual cake made from that recipe            │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────┐      │
│ │ Your Tenant                                                 │      │
│ │ ┌──────────────────┐    ┌──────────────────────────────┐  │      │
│ │ │ App Registration  │───→│ Service Principal             │  │      │
│ │ │ Name: Backend-API │    │ Gets RBAC roles assigned     │  │      │
│ │ │ Client ID: xxx    │    │ Can authenticate with:       │  │      │
│ │ │ Redirect URIs     │    │ ├── Client secret            │  │      │
│ │ │ API permissions   │    │ ├── Certificate              │  │      │
│ │ └──────────────────┘    │ └── Federated credential     │  │      │
│ │                          └──────────────────────────────┘  │      │
│ └────────────────────────────────────────────────────────────┘      │
│                                                                       │
│ When to create App Registration:                                     │
│ ├── Your app needs to authenticate users (OAuth/OIDC)               │
│ ├── Your app needs to call Microsoft Graph API                      │
│ ├── CI/CD pipeline needs Azure access (GitHub, GitLab)              │
│ ├── External service needs Azure access                             │
│ └── Multi-tenant SaaS application                                   │
│                                                                       │
│ When to use Managed Identity instead:                                │
│ ├── Azure resource (VM, App Service, Function) accessing Azure      │
│ ├── No credential management needed                                 │
│ └── ✅ ALWAYS prefer Managed Identity over App Registration         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating an App Registration

```
Portal → Microsoft Entra ID → App registrations → New registration

┌─────────────────────────────────────────────────────────────────┐
│              REGISTER AN APPLICATION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:       [backend-api-app]                                    │
│                                                                   │
│ Supported account types:                                         │
│ ○ Accounts in this org directory only (single-tenant)           │
│   → For internal apps (most common for Azure resource access)   │
│ ○ Accounts in any org directory (multi-tenant)                  │
│   → For SaaS apps used by other organizations                  │
│ ○ Any org directory + personal Microsoft accounts               │
│   → For consumer-facing apps                                    │
│ ○ Personal Microsoft accounts only                              │
│                                                                   │
│ Redirect URI (optional):                                         │
│ Platform: [Web ▼]                                               │
│ URI:      [https://app.techcorp.com/callback]                   │
│                                                                   │
│ [Register]                                                       │
│                                                                   │
│ After creation:                                                   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Application (client) ID: a1b2c3d4-e5f6-g7h8-...          │   │
│ │ Directory (tenant) ID:   x1y2z3w4-a5b6-c7d8-...          │   │
│ │ Object ID:               p1q2r3s4-t5u6-v7w8-...          │   │
│ │                                                            │   │
│ │ Next steps:                                                │   │
│ │ 1. Create client secret or upload certificate             │   │
│ │ 2. Add API permissions if needed                          │   │
│ │ 3. Assign RBAC roles to the service principal             │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Create client secret:                                            │
│ App → Certificates & secrets → New client secret               │
│ Description: [production-key]                                    │
│ Expires:     [24 months ▼]   ⚠️ Max 2 years                    │
│                                                                   │
│ ⚠️ Secret value shown ONCE! Copy immediately!                    │
│ Value: abc123...xyz (this is your password)                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create app registration
  az ad app create --display-name "backend-api-app"

  # Create service principal
  az ad sp create --id APPLICATION_ID

  # Create client secret
  az ad app credential reset --id APPLICATION_ID --years 2

  # Assign RBAC role to service principal
  az role assignment create \
    --assignee SERVICE_PRINCIPAL_ID \
    --role "Contributor" \
    --scope "/subscriptions/SUBSCRIPTION_ID/resourceGroups/rg-prod"
```

---

## Part 5: Managed Identities

### What are Managed Identities?

Managed Identities provide Azure resources with an automatically managed identity in Entra ID. **No credentials to manage** — Azure handles everything.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MANAGED IDENTITIES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two types:                                                           │
│                                                                       │
│ 1. SYSTEM-ASSIGNED                                                   │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── Created WITH the resource (1:1 relationship)             │    │
│ │ ├── Lifecycle tied to resource (delete VM = delete identity) │    │
│ │ ├── Cannot be shared between resources                       │    │
│ │ ├── Identity name = resource name                            │    │
│ │ └── Good for: Single resource needing specific access        │    │
│ │                                                               │    │
│ │ Example: VM "prod-web-01" gets identity "prod-web-01"        │    │
│ │          → Grant this identity access to Key Vault            │    │
│ │          → VM can now read secrets without any stored creds   │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 2. USER-ASSIGNED                                                     │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ├── Created as standalone Azure resource                      │    │
│ │ ├── Lifecycle independent (survives resource deletion)       │    │
│ │ ├── Can be shared across MULTIPLE resources                  │    │
│ │ ├── You choose the name                                      │    │
│ │ └── Good for: Multiple resources needing same permissions    │    │
│ │                                                               │    │
│ │ Example: Identity "id-backend-app"                           │    │
│ │          → Assigned to: VM1, VM2, App Service, Function      │    │
│ │          → All get same access permissions                   │    │
│ │          → Manage RBAC once for all                          │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Supported services (partial list):                                   │
│ ├── Virtual Machines / VMSS                                         │
│ ├── App Service / Functions                                         │
│ ├── Azure Kubernetes Service (AKS)                                  │
│ ├── Logic Apps                                                      │
│ ├── Container Instances                                             │
│ ├── Azure Data Factory                                              │
│ ├── API Management                                                  │
│ └── 50+ services support managed identities                         │
│                                                                       │
│ ✅ ALWAYS use Managed Identity over service principal + secret       │
│    No credentials to store, rotate, or leak!                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Enabling & Using Managed Identity

```
Portal → VM → Identity → System assigned → Status: On

┌─────────────────────────────────────────────────────────────────┐
│ System assigned managed identity                                 │
│ Status: [On]                                                     │
│ Object ID: a1b2c3d4-e5f6-... (auto-generated)                  │
│                                                                   │
│ [Save] → Identity created!                                       │
│                                                                   │
│ Now grant access:                                                │
│ → Key Vault → Access policies → Add → Select principal          │
│   → Search: "prod-web-01" → Select → Grant: Get, List secrets  │
│                                                                   │
│ Or via RBAC:                                                     │
│ → Key Vault → Access control (IAM) → Add role assignment        │
│   → Role: Key Vault Secrets User                                │
│   → Assign to: Managed Identity → prod-web-01                  │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Enable system-assigned identity on VM
  az vm identity assign -g rg-prod -n prod-web-01

  # Create user-assigned identity
  az identity create -g rg-prod -n id-backend-app

  # Assign user-assigned identity to VM
  az vm identity assign -g rg-prod -n prod-web-01 \
    --identities /subscriptions/xxx/resourceGroups/rg-prod/providers/Microsoft.ManagedIdentity/userAssignedIdentities/id-backend-app

  # Grant RBAC to managed identity
  az role assignment create \
    --assignee IDENTITY_PRINCIPAL_ID \
    --role "Key Vault Secrets User" \
    --scope "/subscriptions/xxx/resourceGroups/rg-prod/providers/Microsoft.KeyVault/vaults/prod-kv"

Terraform:
  resource "azurerm_user_assigned_identity" "backend" {
    name                = "id-backend-app"
    resource_group_name = azurerm_resource_group.prod.name
    location            = azurerm_resource_group.prod.location
  }

  resource "azurerm_linux_virtual_machine" "web" {
    # ... other config ...
    identity {
      type         = "UserAssigned"
      identity_ids = [azurerm_user_assigned_identity.backend.id]
    }
  }
```

### Using Managed Identity in Code

```
// C# / .NET (Azure.Identity SDK)
var credential = new DefaultAzureCredential();
var client = new SecretClient(
    new Uri("https://prod-kv.vault.azure.net/"), 
    credential
);
var secret = await client.GetSecretAsync("database-password");

// Python (azure-identity)
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient("https://prod-kv.vault.azure.net/", credential)
secret = client.get_secret("database-password")

// Node.js (@azure/identity)
const { DefaultAzureCredential } = require("@azure/identity");
const { SecretClient } = require("@azure/keyvault-secrets");

const credential = new DefaultAzureCredential();
const client = new SecretClient("https://prod-kv.vault.azure.net/", credential);
const secret = await client.getSecret("database-password");

// DefaultAzureCredential automatically tries (in order):
// 1. Environment variables
// 2. Workload Identity (AKS)
// 3. Managed Identity (VM, App Service)
// 4. Azure CLI (local development)
// 5. Azure PowerShell
// 6. Interactive browser (fallback)
```

---

## Part 6: Azure RBAC (Role-Based Access Control)

### How RBAC Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AZURE RBAC                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ROLE ASSIGNMENT = Security Principal + Role + Scope                   │
│                                                                       │
│ ┌─────────────────┐ ┌─────────────────┐ ┌────────────────────────┐ │
│ │ WHO             │ │ WHAT            │ │ WHERE                   │ │
│ │ (Principal)     │ │ (Role)          │ │ (Scope)                │ │
│ ├─────────────────┤ ├─────────────────┤ ├────────────────────────┤ │
│ │ • User          │ │ • Owner         │ │ • Management Group     │ │
│ │ • Group         │ │ • Contributor   │ │ • Subscription         │ │
│ │ • Service       │ │ • Reader        │ │ • Resource Group       │ │
│ │   Principal     │ │ • Custom role   │ │ • Resource             │ │
│ │ • Managed       │ │ • 100+ built-in │ │                        │ │
│ │   Identity      │ │                 │ │                        │ │
│ └─────────────────┘ └─────────────────┘ └────────────────────────┘ │
│                                                                       │
│ Example: "sg-backend-devs group has Contributor role on               │
│           resource group rg-prod-compute"                            │
│                                                                       │
│ SCOPE HIERARCHY (inheritance):                                       │
│                                                                       │
│ Management Group ─────────────┐                                     │
│    Subscription ────────────┐ │ Role assigned higher                │
│       Resource Group ─────┐ │ │ = applies to all below             │
│          Resource ──────┐ │ │ │                                    │
│                         │ │ │ │                                    │
│ Most specific ←─────────┘ │ │ │                                    │
│ Inherited from above ──────┘ │ │                                    │
│                               │ │                                    │
│                               ▼ ▼                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Built-In Roles

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BUILT-IN ROLES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GENERAL (apply to all resource types):                               │
│ ┌────────────────────────────────────────────────────────────────┐  │
│ │ Role            │ Can do                                        │  │
│ ├────────────────────────────────────────────────────────────────┤  │
│ │ Owner           │ Everything + manage RBAC + assign roles       │  │
│ │ Contributor     │ Everything EXCEPT manage RBAC                 │  │
│ │ Reader          │ Read-only access to all resources             │  │
│ │ User Access     │ Only manage role assignments (no resources)   │  │
│ │   Administrator │                                               │  │
│ └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ COMPUTE:                                                             │
│ ├── Virtual Machine Contributor (manage VMs, not VNet/storage)      │
│ ├── Virtual Machine Administrator Login (admin login to VMs)        │
│ ├── Virtual Machine User Login (user login to VMs)                  │
│ └── Classic Virtual Machine Contributor                              │
│                                                                       │
│ NETWORKING:                                                          │
│ ├── Network Contributor (manage networks, not access)               │
│ ├── DNS Zone Contributor                                            │
│ └── Traffic Manager Contributor                                      │
│                                                                       │
│ STORAGE:                                                             │
│ ├── Storage Account Contributor (manage accounts)                   │
│ ├── Storage Blob Data Contributor (read/write blob data)            │
│ ├── Storage Blob Data Reader (read blob data only)                  │
│ ├── Storage Queue Data Contributor                                  │
│ └── Storage Table Data Contributor                                   │
│                                                                       │
│ DATABASES:                                                           │
│ ├── SQL DB Contributor (manage SQL DBs, not access)                 │
│ ├── SQL Server Contributor (manage SQL servers)                      │
│ ├── Cosmos DB Account Reader                                        │
│ └── Redis Cache Contributor                                         │
│                                                                       │
│ KEY VAULT:                                                           │
│ ├── Key Vault Administrator (full access)                           │
│ ├── Key Vault Secrets User (read secrets only)                      │
│ ├── Key Vault Secrets Officer (manage secrets)                      │
│ ├── Key Vault Certificates Officer                                  │
│ └── Key Vault Crypto User                                           │
│                                                                       │
│ CONTAINERS:                                                          │
│ ├── Azure Kubernetes Service Cluster Admin                          │
│ ├── Azure Kubernetes Service Cluster User                           │
│ ├── AKS RBAC Admin                                                  │
│ └── AKS RBAC Reader                                                 │
│                                                                       │
│ MONITORING:                                                          │
│ ├── Monitoring Contributor                                          │
│ ├── Monitoring Reader                                               │
│ ├── Log Analytics Contributor                                       │
│ └── Application Insights Component Contributor                      │
│                                                                       │
│ SECURITY:                                                            │
│ ├── Security Admin                                                  │
│ ├── Security Reader                                                 │
│ └── Key Vault Administrator                                         │
│                                                                       │
│ ⚠️ There are 400+ built-in roles! These are just the most common.  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Role Assignments

```
Portal → Resource (or RG/Sub) → Access control (IAM) → Add role assignment

┌─────────────────────────────────────────────────────────────────┐
│                ADD ROLE ASSIGNMENT                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Role                                                     │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Search: [Contributor]                                      │   │
│ │                                                            │   │
│ │ ○ Owner                                                    │   │
│ │ ● Contributor (Grants full access but can't assign roles) │   │
│ │ ○ Reader                                                   │   │
│ │ ○ Virtual Machine Contributor                              │   │
│ │ ○ ...                                                      │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 2: Members                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Assign access to: ● User, group, or service principal     │   │
│ │                    ○ Managed identity                       │   │
│ │                                                            │   │
│ │ + Select members: [sg-backend-developers]                  │   │
│ │                                                            │   │
│ │ Selected: sg-backend-developers (12 members)              │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 3: Conditions (optional — for specific data actions)        │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ ○ Not constrained (default)                               │   │
│ │ ○ Constrained (limit to specific conditions)              │   │
│ │   Example: "Only blob containers starting with 'team-x-'" │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Review + assign]                                                │
│                                                                   │
│ Scope: This assignment applies to:                               │
│ /subscriptions/xxx/resourceGroups/rg-prod-compute                │
│ (All resources within this resource group)                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Assign Contributor on resource group
  az role assignment create \
    --assignee "sg-backend-developers" \
    --role "Contributor" \
    --scope "/subscriptions/xxx/resourceGroups/rg-prod-compute"

  # Assign Reader on subscription
  az role assignment create \
    --assignee "user@techcorp.com" \
    --role "Reader" \
    --subscription "Prod-WebApp"

  # Assign to managed identity on specific resource
  az role assignment create \
    --assignee IDENTITY_OBJECT_ID \
    --role "Key Vault Secrets User" \
    --scope "/subscriptions/xxx/resourceGroups/rg-prod/providers/Microsoft.KeyVault/vaults/prod-kv"

  # List role assignments
  az role assignment list --resource-group rg-prod-compute -o table
```

### Custom Roles

```
Portal → Subscriptions → Access control (IAM) → Add → Add custom role

┌─────────────────────────────────────────────────────────────────┐
│                  CREATE CUSTOM ROLE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Custom role name: [VM Operator]                                  │
│ Description:      [Start, stop, restart VMs. No create/delete]  │
│ Baseline:         ○ Start from scratch                          │
│                   ● Clone a role: [Virtual Machine Contributor]  │
│                   ○ Start from JSON                              │
│                                                                   │
│ Permissions:                                                     │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Actions (allowed):                                         │   │
│ │ ├── Microsoft.Compute/virtualMachines/start/action        │   │
│ │ ├── Microsoft.Compute/virtualMachines/powerOff/action     │   │
│ │ ├── Microsoft.Compute/virtualMachines/restart/action      │   │
│ │ ├── Microsoft.Compute/virtualMachines/read                │   │
│ │ └── Microsoft.Compute/virtualMachines/instanceView/read   │   │
│ │                                                            │   │
│ │ NotActions (excluded):                                     │   │
│ │ ├── Microsoft.Compute/virtualMachines/delete              │   │
│ │ ├── Microsoft.Compute/virtualMachines/write               │   │
│ │ └── Microsoft.Compute/virtualMachines/deallocate/action   │   │
│ │                                                            │   │
│ │ DataActions (for data plane):                              │   │
│ │ └── (none for this role)                                  │   │
│ │                                                            │   │
│ │ Assignable scopes:                                         │   │
│ │ └── /subscriptions/xxx (can only assign within this sub)  │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

JSON definition:
{
  "Name": "VM Operator",
  "Description": "Start, stop, restart VMs only",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/powerOff/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/instanceView/read",
    "Microsoft.Resources/subscriptions/resourceGroups/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  ]
}

CLI:
  az role definition create --role-definition @vm-operator-role.json
```

---

## Part 7: Conditional Access

### What is Conditional Access?

Conditional Access policies are **if-then** rules that control how and when users can access resources. They're the backbone of Zero Trust in Azure.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CONDITIONAL ACCESS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ IF (conditions) → THEN (controls)                                    │
│                                                                       │
│ IF:                                  THEN:                           │
│ ├── User/Group                       ├── Block access               │
│ ├── Cloud app                        ├── Grant access with:         │
│ ├── Device platform                  │   ├── Require MFA            │
│ ├── Location (IP/country)            │   ├── Require compliant device│
│ ├── Client app                       │   ├── Require hybrid join    │
│ ├── Sign-in risk                     │   ├── Require app protection │
│ ├── User risk                        │   └── Require password change│
│ └── Device state                     └── Session controls:          │
│                                          ├── Sign-in frequency      │
│                                          ├── Persistent browser      │
│                                          └── App enforced restrictions│
│                                                                       │
│ Requires: Entra ID P1 or P2 license                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Common Conditional Access Policies

```
Portal → Microsoft Entra ID → Security → Conditional Access → New policy

Policy 1: Require MFA for all users
┌─────────────────────────────────────────────────────────────────┐
│ Name: Require MFA for all users                                  │
│ Users: All users (exclude: break-glass accounts)                │
│ Cloud apps: All cloud apps                                      │
│ Conditions: None (always)                                       │
│ Grant: Require multi-factor authentication                      │
│ State: On                                                        │
└─────────────────────────────────────────────────────────────────┘

Policy 2: Block access from untrusted locations
┌─────────────────────────────────────────────────────────────────┐
│ Name: Block untrusted locations                                  │
│ Users: All users                                                │
│ Cloud apps: Azure Management (portal + CLI)                     │
│ Conditions:                                                      │
│   Locations: Exclude named locations (office IPs, VPN)          │
│ Grant: Block access                                             │
│ State: On                                                        │
└─────────────────────────────────────────────────────────────────┘

Policy 3: Require compliant device for production access
┌─────────────────────────────────────────────────────────────────┐
│ Name: Compliant device for production                            │
│ Users: sg-env-prod-access group                                 │
│ Cloud apps: Azure Management                                    │
│ Conditions:                                                      │
│   Device platforms: Any device                                  │
│ Grant: Require device to be marked as compliant                 │
│ State: On                                                        │
│                                                                   │
│ (Devices must be enrolled in Intune and meet compliance policy) │
└─────────────────────────────────────────────────────────────────┘

Policy 4: Require MFA for risky sign-ins
┌─────────────────────────────────────────────────────────────────┐
│ Name: MFA for risky sign-ins                                     │
│ Users: All users                                                │
│ Cloud apps: All cloud apps                                      │
│ Conditions:                                                      │
│   Sign-in risk: Medium, High                                    │
│ Grant: Require multi-factor authentication                      │
│ State: On                                                        │
│                                                                   │
│ (Requires P2 — Identity Protection detects risky sign-ins)      │
└─────────────────────────────────────────────────────────────────┘

⚠️ ALWAYS exclude break-glass accounts from all policies!
   These are emergency-access accounts that bypass Conditional Access.
   They should have: very long complex passwords, MFA, monitored alerts.
```

---

## Part 8: Privileged Identity Management (PIM)

### What is PIM?

PIM provides **just-in-time** privileged access — users don't have permanent admin roles. They activate roles when needed, for a limited time.

```
┌─────────────────────────────────────────────────────────────────────┐
│              PRIVILEGED IDENTITY MANAGEMENT (PIM)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without PIM:                                                         │
│ John has "Contributor" permanently                                   │
│ → If compromised, attacker has Contributor access 24/7               │
│                                                                       │
│ With PIM:                                                            │
│ John is "eligible" for Contributor                                   │
│ → Must explicitly activate (with justification)                     │
│ → Gets access for 4 hours                                           │
│ → Requires approval for sensitive roles                              │
│ → All activations logged and auditable                              │
│                                                                       │
│ Assignment types:                                                    │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │ ELIGIBLE:                                                    │    │
│ │ ├── User CAN activate the role when needed                  │    │
│ │ ├── Must go through activation workflow                     │    │
│ │ ├── Time-limited when activated (e.g., 4 hours)            │    │
│ │ ├── Can require: MFA, justification, approval              │    │
│ │ └── ✅ USE THIS for almost all privileged roles              │    │
│ │                                                              │    │
│ │ ACTIVE:                                                      │    │
│ │ ├── User HAS the role permanently (or for set duration)    │    │
│ │ ├── No activation needed                                    │    │
│ │ ├── Use sparingly (break-glass, service principals)        │    │
│ │ └── ⚠️ Increases attack surface                             │    │
│ └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Applies to:                                                          │
│ ├── Entra ID roles (Global Admin, User Admin, etc.)                 │
│ ├── Azure resource roles (Owner, Contributor at any scope)          │
│ └── Groups (PIM for Groups — activate group membership)             │
│                                                                       │
│ Requires: Entra ID P2 license                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Activation flow:
1. User → PIM → "Activate" Contributor role on rg-prod
2. Provide justification: "Deploying hotfix for incident #1234"
3. Requires MFA verification
4. Approval needed (if configured) → Approver notified
5. Approved → Role active for 4 hours
6. After 4 hours → Auto-deactivated
7. All actions during activation window → logged in audit trail
```

---

## Part 9: Multi-Factor Authentication (MFA)

```
┌─────────────────────────────────────────────────────────────────┐
│                        MFA OPTIONS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Methods (strongest → weakest):                                   │
│ ├── FIDO2 Security Key (passwordless, phishing-resistant)       │
│ │   └── YubiKey, Windows Hello                                 │
│ ├── Microsoft Authenticator (push notification + number match)  │
│ │   └── ✅ Recommended for most users                          │
│ ├── Windows Hello for Business (biometric/PIN)                  │
│ ├── Certificate-based authentication                            │
│ ├── OATH hardware tokens (TOTP)                                 │
│ ├── OATH software tokens (Authenticator apps)                   │
│ ├── SMS (one-time code via text)                                │
│ │   └── ⚠️ Least secure (SIM swap attacks)                     │
│ └── Voice call                                                  │
│     └── ⚠️ Least secure                                        │
│                                                                   │
│ Enable MFA:                                                      │
│ ├── Method 1: Conditional Access (recommended - granular)       │
│ │   Create policy: All users → All apps → Require MFA          │
│ │                                                                │
│ ├── Method 2: Security Defaults (free, basic)                   │
│ │   Entra ID → Properties → Security defaults → Enabled        │
│ │   Requires MFA for admins, MFA for all users when needed     │
│ │                                                                │
│ └── Method 3: Per-user MFA (legacy — avoid)                     │
│     Entra ID → Users → Per-user MFA                             │
│                                                                   │
│ Number matching (enabled by default now):                        │
│ Sign-in shows: "Enter the number shown: 42"                    │
│ Phone shows: "42" → user enters "42" to approve                │
│ Prevents MFA fatigue attacks (blindly approving)                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Entra ID Roles vs Azure RBAC Roles

```
⚠️ These are TWO DIFFERENT ROLE SYSTEMS!

┌─────────────────────────────────────────────────────────────────────┐
│          ENTRA ID ROLES vs AZURE RBAC ROLES                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ENTRA ID (Directory) ROLES:                                          │
│ ├── Manage Entra ID objects (users, groups, apps)                   │
│ ├── Scope: Entire tenant (or Administrative Units)                  │
│ ├── Examples:                                                        │
│ │   ├── Global Administrator (full control of tenant)               │
│ │   ├── User Administrator (manage users)                           │
│ │   ├── Groups Administrator (manage groups)                        │
│ │   ├── Application Administrator (manage app registrations)        │
│ │   ├── Security Administrator (manage security features)           │
│ │   ├── Billing Administrator (manage billing)                      │
│ │   ├── Helpdesk Administrator (reset passwords)                    │
│ │   └── Conditional Access Administrator                            │
│ ├── Assigned via: Entra ID → Roles and administrators              │
│ └── NOT for Azure resources!                                        │
│                                                                       │
│ AZURE RBAC ROLES:                                                    │
│ ├── Manage Azure resources (VMs, storage, databases)                │
│ ├── Scope: Management Group / Subscription / RG / Resource          │
│ ├── Examples:                                                        │
│ │   ├── Owner (full resource access + RBAC management)              │
│ │   ├── Contributor (full resource access, no RBAC)                 │
│ │   ├── Reader (read-only)                                          │
│ │   ├── Virtual Machine Contributor                                 │
│ │   ├── Storage Blob Data Reader                                    │
│ │   └── Key Vault Secrets User                                      │
│ ├── Assigned via: Resource → Access control (IAM)                   │
│ └── NOT for managing users/groups/tenant!                           │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │ "I want to create a new user"                                │    │
│ │  → Need: Entra ID role (User Administrator)                 │    │
│ │                                                              │    │
│ │ "I want to create a virtual machine"                         │    │
│ │  → Need: Azure RBAC role (Virtual Machine Contributor)      │    │
│ │                                                              │    │
│ │ "I want to manage everything"                                │    │
│ │  → Need: Global Admin (Entra ID) + Owner (Azure RBAC)      │    │
│ └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Note: Global Administrator CAN elevate to User Access Administrator  │
│ on Azure root scope (Entra ID → Properties → Access management for  │
│ Azure resources). This bridges both systems.                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Security Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│              IDENTITY & ACCESS BEST PRACTICES                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. ENFORCE MFA FOR EVERYONE                                         │
│  ├── Conditional Access policy: All users → Require MFA             │
│  ├── Exclude break-glass accounts (but alert on their use)          │
│  ├── Use Microsoft Authenticator + number matching                   │
│  └── Phishing-resistant MFA (FIDO2) for admins                      │
│                                                                       │
│  2. LEAST PRIVILEGE                                                  │
│  ├── Use PIM for privileged roles (no permanent admin)              │
│  ├── Assign RBAC at narrowest scope (resource > RG > sub)           │
│  ├── Prefer predefined roles over Owner/Contributor                 │
│  ├── Regular access reviews (quarterly)                             │
│  └── Use Azure RBAC recommendations                                │
│                                                                       │
│  3. USE MANAGED IDENTITIES                                           │
│  ├── Never store credentials in code/config                         │
│  ├── Managed Identity for all Azure-to-Azure communication          │
│  ├── Workload Identity Federation for CI/CD                         │
│  └── Rotate app registration secrets before expiry                  │
│                                                                       │
│  4. USE GROUPS (not individual assignments)                          │
│  ├── Dynamic groups for team membership                             │
│  ├── Security groups for RBAC assignments                           │
│  ├── When person changes team → group auto-updates                  │
│  └── Easier to audit and review                                     │
│                                                                       │
│  5. CONDITIONAL ACCESS                                               │
│  ├── Block legacy authentication protocols                          │
│  ├── Require compliant devices for sensitive apps                   │
│  ├── Restrict access by location for admin portals                  │
│  ├── Require MFA for risky sign-ins                                 │
│  └── Session lifetime limits for sensitive resources                │
│                                                                       │
│  6. MONITORING & RESPONSE                                            │
│  ├── Enable Entra ID sign-in logs (sent to Log Analytics)           │
│  ├── Alert on: Global Admin activation, new app registration,      │
│  │   consent grants, impossible travel, MFA fatigue attacks         │
│  ├── Identity Protection: Auto-remediate risky users                │
│  └── Regular access reviews via Entra ID Governance                 │
│                                                                       │
│  7. BREAK-GLASS ACCOUNTS                                             │
│  ├── 2 emergency-access accounts (cloud-only, not synced)           │
│  ├── No MFA or only FIDO2 key (stored in safe)                      │
│  ├── Excluded from all Conditional Access policies                  │
│  ├── Global Admin role (permanently assigned)                       │
│  ├── Long, complex passwords in physical safe                       │
│  └── Alert immediately on any use                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Real-World Patterns

### Startup (5-10 Developers)

```
Setup:
├── 1 Entra ID tenant (techcorp.onmicrosoft.com + techcorp.com verified)
├── Security Defaults enabled (free MFA)
├── Users created directly in Entra ID
├── Groups:
│   ├── sg-admins (2-3 people — Contributor on subscription)
│   ├── sg-developers (all devs — Contributor on dev RG, Reader on prod)
│   └── sg-all-users (dynamic: all members)
│
├── Managed Identities:
│   ├── App Service → system-assigned → Key Vault + SQL access
│   └── VM → system-assigned → Storage access
│
├── App Registrations:
│   └── GitHub Actions (federated credential, no secrets)
│
└── Cost: Free tier (Security Defaults covers MFA)
```

### Mid-Size Company (50-100 Developers)

```
Setup:
├── Entra ID P1 license
├── Entra Connect (sync from on-premises AD)
├── Custom domain: techcorp.com
│
├── Groups (dynamic + assigned):
│   ├── sg-team-backend (dynamic: department=Engineering AND title contains Backend)
│   ├── sg-team-frontend (dynamic)
│   ├── sg-team-data (dynamic)
│   ├── sg-role-prod-contributor (assigned — vetted members only)
│   ├── sg-role-prod-reader (dynamic: all engineering)
│   ├── sg-role-sql-admin (assigned: DBAs)
│   └── sg-external-contractors (guest users)
│
├── RBAC:
│   ├── sg-role-prod-contributor → Contributor on Production RGs
│   ├── sg-role-prod-reader → Reader on Production subscription
│   ├── sg-team-* → Contributor on Dev/Staging RGs
│   └── sg-role-sql-admin → SQL DB Contributor on data RGs
│
├── Conditional Access:
│   ├── All users → Require MFA (always)
│   ├── Admin portals → Block outside office IPs
│   ├── Contractors → Block outside business hours
│   └── Legacy auth → Block everywhere
│
├── Managed Identities:
│   ├── User-assigned: id-backend-app (shared by App Services)
│   ├── User-assigned: id-data-pipeline (Data Factory + storage)
│   └── System-assigned: Each AKS cluster
│
└── Monitoring:
    ├── Sign-in logs → Log Analytics
    └── Alerts on admin role activation, failed MFA
```

### Enterprise (500+ Developers)

```
Setup:
├── Entra ID P2 license
├── Entra Connect with password hash sync + SSO
├── Multiple verified domains
│
├── Privileged Identity Management (PIM):
│   ├── All admin roles are ELIGIBLE only (no permanent admins)
│   ├── Activation requires: MFA + justification + approval
│   ├── Max duration: 4 hours (8 for on-call)
│   ├── Monthly access reviews by managers
│   └── Break-glass: 2 accounts with permanent Global Admin
│
├── Conditional Access (15+ policies):
│   ├── Baseline: MFA for everyone
│   ├── Admin: Compliant device + trusted location + MFA
│   ├── High-risk: Block or password change
│   ├── Medium-risk: Require MFA
│   ├── Guest: Limited app access + MFA
│   ├── Device: Must be Intune compliant
│   └── Session: 8-hour max for sensitive apps
│
├── Identity Governance:
│   ├── Access reviews: Quarterly for all privileged roles
│   ├── Entitlement management: Access packages for projects
│   ├── Lifecycle workflows: Auto-onboard/offboard
│   └── B2B: Automatic guest expiration (90 days)
│
├── Administrative Units (delegate user management):
│   ├── AU: India Office → HR India manages users
│   ├── AU: US Office → HR US manages users
│   └── AU: Contractors → Vendor management team
│
├── Advanced security:
│   ├── Identity Protection: Auto-remediate risky users
│   ├── Token protection: Require token binding
│   ├── Continuous Access Evaluation (CAE)
│   └── Global Secure Access (ZTNA replacement for VPN)
│
└── Compliance:
    ├── SOC2: Access reviews + audit logs + PIM
    ├── ISO 27001: Documented IAM procedures
    ├── Audit logs retained 2 years (Log Analytics)
    └── External audit: Annual IAM configuration review
```

---

## Quick Reference: CLI Commands

```bash
# Users
az ad user list --query "[].{Name:displayName, UPN:userPrincipalName}" -o table
az ad user create --display-name "Name" --user-principal-name "u@domain" --password "Pass"
az ad user delete --id USER_OBJECT_ID

# Groups
az ad group list --query "[].{Name:displayName, ID:id}" -o table
az ad group create --display-name "sg-name" --mail-nickname "sg-name"
az ad group member add --group "sg-name" --member-id USER_OBJECT_ID
az ad group member list --group "sg-name" -o table

# App Registrations
az ad app list --query "[].{Name:displayName, AppId:appId}" -o table
az ad app create --display-name "app-name"
az ad sp create --id APP_ID

# Managed Identities
az identity create -g RG_NAME -n IDENTITY_NAME
az identity list -g RG_NAME -o table

# RBAC
az role assignment list --resource-group RG_NAME -o table
az role assignment create --assignee PRINCIPAL --role "Role Name" --scope SCOPE
az role assignment delete --assignee PRINCIPAL --role "Role Name" --scope SCOPE
az role definition list --query "[?contains(roleName,'Contributor')]" -o table
az role definition create --role-definition @custom-role.json

# Service Principals
az ad sp list --display-name "name" -o table
az ad sp credential reset --id SP_ID --years 2
```

---

## What's Next?

In the next chapter, we'll cover Azure Billing & Cost Management — pricing models, Azure Cost Management, budgets, Reserved Instances, Azure Hybrid Benefit, and cost optimization.

→ Next: [Chapter 5: Billing & Cost Management](05-billing-and-cost-management.md)

---

*Last Updated: May 2026*
