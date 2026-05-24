# Chapter 42: Microsoft Entra ID (Azure Active Directory)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Entra ID Fundamentals](#part-1-entra-id-fundamentals)
- [Part 2: Managing Users & Groups (Portal Walkthrough)](#part-2-managing-users--groups-portal-walkthrough)
- [Part 3: Application Registration](#part-3-application-registration)
- [Part 4: Conditional Access](#part-4-conditional-access)
- [Part 5: Multi-Factor Authentication (MFA)](#part-5-multi-factor-authentication-mfa)
- [Part 6: Enterprise Applications & SSO](#part-6-enterprise-applications--sso)
- [Part 7: Managed Identities](#part-7-managed-identities)
- [Part 8: Terraform & az CLI Reference](#part-8-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Microsoft Entra ID (formerly Azure Active Directory / Azure AD) is Azure's cloud identity and access management service. It handles who you are (authentication) and what you can access (authorization). Every Azure subscription has an Entra ID tenant — it's the foundation of all security in Azure.

```
What you'll learn:
├── Entra ID Fundamentals
│   ├── What is Entra ID (cloud identity provider)
│   ├── Tenant, Directory, Users, Groups
│   ├── Entra ID vs on-prem Active Directory
│   └── Free vs Premium tiers
├── Managing Users & Groups (Portal)
├── Application Registration (apps accessing Azure)
├── Conditional Access (smart security policies)
├── Multi-Factor Authentication (MFA)
├── Enterprise Applications & SSO
├── Managed Identities (passwordless for Azure resources)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Entra ID Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           ENTRA ID OVERVIEW                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Entra ID?                                                    │
│ ├── Cloud-based identity provider                                │
│ ├── Handles authentication (proving who you are)                │
│ ├── Handles authorization (what you can access)                 │
│ ├── Single Sign-On (SSO) to 1000s of apps                      │
│ └── Used by Azure, Microsoft 365, and your custom apps         │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Tenant: Your organization's instance of Entra ID           │
│ │   (e.g., contoso.onmicrosoft.com)                              │
│ ├── Directory: Contains users, groups, apps                      │
│ ├── Users: People or service accounts                            │
│ ├── Groups: Collections of users (for permissions)              │
│ ├── App Registrations: Apps that use Entra ID for auth         │
│ └── Service Principals: App identities in your tenant          │
│                                                                       │
│ Entra ID vs On-Prem Active Directory:                               │
│ ┌──────────────────┬────────────────────┬──────────────────┐     │
│ │ Feature          │ On-Prem AD         │ Entra ID         │     │
│ ├──────────────────┼────────────────────┼──────────────────┤     │
│ │ Type             │ Directory service  │ Identity platform│     │
│ │ Protocol         │ LDAP, Kerberos     │ OAuth, SAML, OIDC│     │
│ │ Location         │ Your servers       │ Cloud (Microsoft)│     │
│ │ Authentication   │ Username/password  │ MFA, passwordless│     │
│ │ Devices          │ Domain join        │ Azure AD join    │     │
│ │ Apps             │ On-prem apps       │ Cloud + SaaS     │     │
│ │ Group Policy     │ Yes (GPO)          │ Intune/Conditional│    │
│ │ Hybrid           │ AD Connect sync    │ Entra Connect    │     │
│ └──────────────────┴────────────────────┴──────────────────┘     │
│                                                                       │
│ Tiers:                                                               │
│ ├── Free: Basic users, groups, SSO (included with Azure)        │
│ ├── P1 ($6/user/mo): Conditional Access, dynamic groups        │
│ └── P2 ($9/user/mo): Identity Protection, PIM, access reviews │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Managing Users & Groups (Portal Walkthrough)

```
Console → Microsoft Entra ID → Users

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE USER                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ [+ New user] → Create new user                                     │
│                                                                       │
│ Identity:                                                            │
│ User principal name: [john.doe] @contoso.onmicrosoft.com          │
│ Display name: [John Doe]                                           │
│ First name: [John]                                                  │
│ Last name: [Doe]                                                    │
│                                                                       │
│ Password:                                                            │
│ ● Auto-generate  ○ Let me create                                 │
│ ☑ Require password change at first sign-in                       │
│                                                                       │
│ Properties (optional):                                              │
│ Job title: [Software Engineer]                                     │
│ Department: [Engineering]                                          │
│ Usage location: [India ▼] (required for licensing!)              │
│                                                                       │
│ Groups: [+ Add groups]                                              │
│ Roles: [+ Add role] (e.g., Global Administrator)                 │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ OR invite external user:                                            │
│ [+ New user] → Invite external user                               │
│ Email: [partner@external.com]                                     │
│ ⚡ B2B collaboration — external user signs in with their own ID │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE GROUP                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Entra ID → Groups → [+ New group]                                │
│                                                                       │
│ Group type: [Security ▼]                                          │
│ ├── Security: For managing access to resources                   │
│ └── Microsoft 365: For collaboration (email, Teams, SharePoint)│
│                                                                       │
│ Group name: [sg-developers]                                        │
│ Description: [All developers in engineering team]                │
│                                                                       │
│ Membership type: [Assigned ▼]                                     │
│ ├── Assigned: Manually add/remove members                       │
│ ├── Dynamic User: Auto-add based on attributes (P1 required)  │
│ │   Rule: user.department -eq "Engineering"                     │
│ │   ⚡ Users in Engineering dept automatically added!          │
│ └── Dynamic Device: Auto-add devices based on properties       │
│                                                                       │
│ Members: [+ Add members]                                          │
│ Owner: [admin@contoso.onmicrosoft.com]                            │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Application Registration

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP REGISTRATION                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Entra ID → App registrations → [+ New registration]             │
│                                                                       │
│ Name: [MyApp API]                                                   │
│ Supported account types:                                            │
│ ● Single tenant (this org only)                                   │
│ ○ Multitenant (any org)                                          │
│ ○ Multitenant + personal Microsoft accounts                     │
│                                                                       │
│ Redirect URI: [Web ▼] https://myapp.com/auth/callback           │
│                                                                       │
│ [Register]                                                           │
│                                                                       │
│ After registration you get:                                         │
│ ├── Application (client) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx│
│ ├── Directory (tenant) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
│ └── Client secret: Create under "Certificates & secrets"       │
│                                                                       │
│ What you can do with App Registration:                              │
│ ├── OAuth 2.0 / OpenID Connect authentication                  │
│ ├── API permissions (access Microsoft Graph, custom APIs)       │
│ ├── Expose an API (define scopes for your own API)             │
│ ├── Certificates & secrets (authentication credentials)        │
│ └── Token configuration (customize JWT claims)                  │
│                                                                       │
│ Common use cases:                                                    │
│ ├── Web app login with "Sign in with Microsoft"                 │
│ ├── API-to-API authentication (client credentials flow)        │
│ ├── SPA (Single Page App) authentication (auth code + PKCE)   │
│ └── CI/CD pipeline authentication (service principal)          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Conditional Access

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONDITIONAL ACCESS (P1 required)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Entra ID → Security → Conditional Access → [+ New policy]      │
│                                                                       │
│ Name: [Require MFA for admins]                                     │
│                                                                       │
│ Assignments:                                                         │
│ Users: ☑ Directory roles → Global Administrator, User Admin    │
│ Cloud apps: ☑ All cloud apps                                     │
│ Conditions:                                                          │
│   Locations: All locations (except trusted office IPs)           │
│   Device platforms: Any                                            │
│   Client apps: Browser + Mobile apps                              │
│   Sign-in risk: Medium and above (P2)                            │
│                                                                       │
│ Access controls:                                                     │
│ Grant:                                                               │
│ ☑ Require multi-factor authentication                            │
│ ☑ Require compliant device                                       │
│ ☐ Require hybrid Azure AD joined device                         │
│ ☐ Require approved client app                                   │
│                                                                       │
│ Session:                                                             │
│ Sign-in frequency: [Every 1 hour ▼]                              │
│ Persistent browser session: Disabled                              │
│                                                                       │
│ Enable policy: ● On  ○ Off  ○ Report-only                     │
│                                                                       │
│ [Create]                                                             │
│                                                                       │
│ Common policies:                                                     │
│ ├── Require MFA for all admins                                   │
│ ├── Require MFA for all users (start with report-only!)        │
│ ├── Block legacy authentication (IMAP, POP3, SMTP)             │
│ ├── Require compliant device for sensitive apps                 │
│ ├── Block access from certain countries                         │
│ └── Require MFA for risky sign-ins (P2)                        │
│                                                                       │
│ ⚡ IF (condition) THEN (access control)                          │
│ "IF admin user THEN require MFA"                                 │
│ "IF sign-in from unknown location THEN block"                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Multi-Factor Authentication (MFA)

```
MFA = Something you know + Something you have

Methods:
├── Microsoft Authenticator app (push notification) ← Recommended
├── FIDO2 security key (hardware key)
├── Windows Hello for Business
├── SMS verification (less secure, phishing risk)
├── Voice call
└── Software OATH tokens

Enable MFA:
├── Per-user MFA (legacy): Entra ID → Users → Per-user MFA
├── Conditional Access (recommended): Create policy requiring MFA
├── Security Defaults (free, simple): Entra ID → Properties → Security Defaults → Enable
│   ⚡ Security Defaults = Enable MFA for all users with minimal config
│   ⚡ Great for small organizations
└── Passwordless: Authenticator app, FIDO2, Windows Hello
```

---

## Part 6: Enterprise Applications & SSO

```
Entra ID → Enterprise applications

┌─────────────────────────────────────────────────────────────────────┐
│           ENTERPRISE APPLICATIONS & SSO                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Enterprise Applications?                                    │
│ ├── Pre-integrated SaaS apps (1000s in gallery)                 │
│ ├── Configure Single Sign-On (SSO) for your organization       │
│ └── Manage who can access which apps                            │
│                                                                       │
│ Add app: [+ New application]                                       │
│ Search gallery: [Salesforce ▼]                                    │
│                                                                       │
│ SSO setup:                                                           │
│ Enterprise app → Single sign-on → [SAML ▼]                     │
│ ├── SAML: Most enterprise apps (Salesforce, ServiceNow)        │
│ ├── OIDC: Modern apps (OAuth 2.0 / OpenID Connect)            │
│ ├── Password-based: Store credentials in Entra (less secure)  │
│ └── Linked: Just redirect to app's login page                  │
│                                                                       │
│ User assignment:                                                     │
│ Enterprise app → Users and groups → [+ Add user/group]         │
│ ├── Assign specific users or groups                              │
│ └── ☑ Assignment required: Only assigned users can access     │
│                                                                       │
│ Result: Users see the app in My Apps portal (myapps.microsoft.com)│
│ One click → Automatically signed in (SSO) ✅                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Managed Identities

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGED IDENTITIES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Your app needs to access Azure resources (Key Vault, SQL, │
│ Storage). How to authenticate without storing passwords in code? │
│                                                                       │
│ Solution: Managed Identity = Azure gives your resource an identity│
│ that automatically authenticates. No passwords, no secrets!      │
│                                                                       │
│ Types:                                                               │
│ ├── System-assigned:                                              │
│ │   ├── Created with the resource (1:1 relationship)           │
│ │   ├── Deleted when resource is deleted                        │
│ │   ├── Enable: Resource → Identity → System assigned → On  │
│ │   └── Use case: Single resource needs access to something   │
│ │                                                                  │
│ └── User-assigned:                                                │
│     ├── Created independently as a separate Azure resource     │
│     ├── Can be assigned to MULTIPLE resources                  │
│     ├── Persists when resource is deleted                       │
│     └── Use case: Multiple resources share same identity       │
│                                                                       │
│ Example: App Service accessing Key Vault                          │
│ 1. Enable managed identity on App Service                        │
│ 2. Give that identity "Key Vault Secrets User" role on Key Vault│
│ 3. App Service can now read secrets — no password needed! ✅    │
│                                                                       │
│ Supported Azure services:                                           │
│ ├── VMs, App Service, Functions, AKS, Container Instances       │
│ ├── Logic Apps, Data Factory, API Management                    │
│ └── Almost every Azure compute service                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & az CLI Reference

### Terraform

```hcl
# Create user
resource "azuread_user" "developer" {
  user_principal_name = "john.doe@contoso.onmicrosoft.com"
  display_name        = "John Doe"
  password            = var.initial_password
}

# Create group
resource "azuread_group" "developers" {
  display_name     = "sg-developers"
  security_enabled = true
  members          = [azuread_user.developer.object_id]
}

# App registration
resource "azuread_application" "myapp" {
  display_name = "MyApp API"
}

resource "azuread_service_principal" "myapp" {
  client_id = azuread_application.myapp.client_id
}
```

### Bicep

```bicep
// Note: Entra ID (Azure AD) resources are managed via Microsoft Graph API
// Bicep can create Managed Identities and role assignments:

// User-assigned managed identity
resource managedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-myapp-prod'
  location: resourceGroup().location
}

// Role assignment (assign identity to a role)
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, managedIdentity.id, 'Reader')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'acdd72a7-3385-48ef-bd42-f606fba81ae7') // Reader
    principalId: managedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}
```

```bash
# Create user
az ad user create \
  --display-name "John Doe" \
  --user-principal-name john.doe@contoso.onmicrosoft.com \
  --password "TempP@ssw0rd!" \
  --force-change-password-next-sign-in true

# List users
az ad user list --output table

# Create group
az ad group create --display-name sg-developers --mail-nickname sg-developers

# Add user to group
az ad group member add --group sg-developers --member-id <user-object-id>

# Create app registration
az ad app create --display-name "MyApp API"

# Create service principal for app
az ad sp create --id <app-id>

# Delete user
az ad user delete --id john.doe@contoso.onmicrosoft.com
```

---

## Real-World Patterns

### Pattern 1: Enterprise SSO with Conditional Access

```
┌─────────────────────────────────────────────────┐
│           Enterprise SSO Architecture           │
├─────────────────────────────────────────────────┤
│                                                 │
│  User Login ──→ Entra ID ──→ Conditional Access │
│                    │              │              │
│                    │         ┌────┴────┐         │
│                    │         │ Policies │         │
│                    │         ├─────────┤         │
│                    │         │ MFA?    │         │
│                    │         │ Device? │         │
│                    │         │ Location│         │
│                    │         └────┬────┘         │
│                    ▼              ▼              │
│              ┌──────────┐  ┌──────────┐         │
│              │ App 1    │  │ App 2    │         │
│              │ (SaaS)   │  │ (Custom) │         │
│              └──────────┘  └──────────┘         │
│                                                 │
│  Setup Steps:                                   │
│  1. Register apps in App Registrations          │
│  2. Configure Enterprise Applications           │
│  3. Create Conditional Access policies          │
│  4. Require MFA for sensitive apps              │
│  5. Block access from untrusted locations       │
└─────────────────────────────────────────────────┘
```

### Pattern 2: B2C Customer Identity for Web App

```
┌─────────────────────────────────────────────────┐
│        B2C Customer Identity Flow               │
├─────────────────────────────────────────────────┤
│                                                 │
│  Customer ──→ Azure AD B2C Tenant               │
│                    │                            │
│              ┌─────┴─────┐                      │
│              │ User Flow │                      │
│              ├───────────┤                      │
│              │ Sign Up   │                      │
│              │ Sign In   │                      │
│              │ Profile   │                      │
│              │ Password  │                      │
│              └─────┬─────┘                      │
│                    │                            │
│         ┌──────────┼──────────┐                 │
│         ▼          ▼          ▼                 │
│     Google      Facebook    Email               │
│     Login       Login       Login               │
│                                                 │
│  Use Case: E-commerce site with social login    │
│  - Millions of customer accounts                │
│  - Self-service password reset                  │
│  - Custom branding on login pages               │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Entra ID = Cloud identity provider (formerly Azure AD)
Tenant = Your organization's Entra ID instance

Users: People + Service accounts (internal + B2B guests)
Groups: Security groups (RBAC) + Microsoft 365 groups
App Registrations: Register apps for OAuth/OIDC auth
Service Principals: App identities in your tenant

Conditional Access (P1): IF condition THEN access control
  Example: IF admin THEN require MFA
MFA: Authenticator app (recommended) + FIDO2 + SMS
Security Defaults: Free MFA for everyone (enable first!)

Managed Identities: Passwordless auth for Azure resources
  System-assigned: 1:1 with resource, auto-deleted
  User-assigned: Separate resource, shared across many

Enterprise Apps: SSO to 1000s of SaaS apps (SAML/OIDC)
Tiers: Free | P1 ($6/user) | P2 ($9/user)
```

---

## What's Next?

Next chapter: [Chapter 43: Key Vault](43-key-vault.md) — Securely store and manage secrets, keys, and certificates.
