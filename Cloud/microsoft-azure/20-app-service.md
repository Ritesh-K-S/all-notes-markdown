# Chapter 20: Azure App Service

---

## Table of Contents

- [Overview](#overview)
- [Part 1: App Service Fundamentals](#part-1-app-service-fundamentals)
- [Part 2: Creating a Web App (Full Portal Walkthrough)](#part-2-creating-a-web-app-full-portal-walkthrough)
- [Part 3: App Service Plans (Deep Dive)](#part-3-app-service-plans-deep-dive)
- [Part 4: Deployment Slots](#part-4-deployment-slots)
- [Part 5: Custom Domains & SSL](#part-5-custom-domains--ssl)
- [Part 6: Scaling (Up & Out)](#part-6-scaling-up--out)
- [Part 7: Configuration & App Settings](#part-7-configuration--app-settings)
- [Part 8: Deployment Methods](#part-8-deployment-methods)
- [Part 9: Networking & Access Restrictions](#part-9-networking--access-restrictions)
- [Part 10: Terraform & Bicep](#part-10-terraform--bicep)
- [Part 11: az CLI Reference](#part-11-az-cli-reference)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure App Service is a fully managed platform (PaaS) for building, deploying, and scaling web apps. You just bring your code — Azure handles the servers, OS patches, load balancing, and scaling. It supports .NET, Java, Node.js, Python, PHP, and Ruby.

Think of it like this: Instead of renting a server (VM), installing the OS, installing a web server (Nginx/IIS), deploying your code, and managing everything — App Service does ALL of that for you. You just upload your code and it runs.

```
What you'll learn:
├── App Service Fundamentals
│   ├── What is App Service (PaaS for web apps)
│   ├── App Service Plan (the compute behind your app)
│   ├── Web App vs API App vs Mobile App
│   └── App Service vs VMs vs Functions vs Container Apps
├── Creating a Web App (Full Portal Walkthrough)
│   ├── Basics (name, runtime, region, plan)
│   ├── Deployment (CI/CD, GitHub Actions)
│   ├── Networking (VNet integration, access restrictions)
│   ├── Monitoring (Application Insights)
│   └── Tags
├── App Service Plans (Pricing Tiers)
│   ├── Free & Shared (F1, D1)
│   ├── Basic (B1, B2, B3)
│   ├── Standard (S1, S2, S3)
│   ├── Premium v3 (P0v3, P1v3, P2v3, P3v3)
│   └── Isolated v2 (I1v2, I2v2, I3v2)
├── Deployment Slots (staging/prod swap)
├── Custom Domains & SSL Certificates
├── Scaling (manual, auto-scale rules)
├── Configuration & App Settings
├── Deployment Methods (Git, GitHub, ZIP, Docker)
├── Networking (VNet integration, private endpoints)
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: App Service Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP SERVICE CONCEPT                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is App Service?                                                │
│ ├── Fully managed PaaS (Platform as a Service)                  │
│ ├── You bring CODE → Azure runs it (no server management!)    │
│ ├── Built-in load balancing, auto-scaling, SSL                 │
│ ├── Multiple language runtimes (.NET, Node, Python, Java, PHP) │
│ ├── Built-in CI/CD (GitHub Actions, Azure DevOps)              │
│ └── Features: deployment slots, custom domains, auth, backups  │
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ App Service Plan (the "server" - compute resources)          │  │
│ │ ┌────────────────────────────────────────────────────────┐  │  │
│ │ │ Region: Central India                                  │  │  │
│ │ │ OS: Linux                                              │  │  │
│ │ │ SKU: P1v3 (2 vCPU, 8 GB RAM)                         │  │  │
│ │ │ Instances: 3 (auto-scaled)                            │  │  │
│ │ │                                                        │  │  │
│ │ │ ┌──────────┐ ┌──────────┐ ┌──────────┐              │  │  │
│ │ │ │ Web App 1│ │ Web App 2│ │ Web App 3│              │  │  │
│ │ │ │ Frontend │ │ API      │ │ Admin    │              │  │  │
│ │ │ │ (React)  │ │ (Node.js)│ │ (Python) │              │  │  │
│ │ │ └──────────┘ └──────────┘ └──────────┘              │  │  │
│ │ │                                                        │  │  │
│ │ │ ⚡ Multiple apps can share ONE plan!                   │  │  │
│ │ │    All share the same compute (CPU/RAM).              │  │  │
│ │ └────────────────────────────────────────────────────────┘  │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Key concepts:                                                        │
│ ├── App Service Plan = The compute (CPU, RAM, instances)        │
│ │   Think of it as the "server" your apps run on.              │
│ │   You pay for the plan, not per app.                         │
│ ├── Web App = Your application deployed on the plan             │
│ │   Multiple apps can share one plan.                          │
│ ├── Deployment Slot = A copy of your app (for staging/testing) │
│ │   Swap slots to deploy with ZERO downtime.                   │
│ └── Custom Domain = Use your own domain (myapp.com)            │
│     Free SSL with Azure managed certificates.                  │
│                                                                       │
│ App Service types:                                                   │
│ ├── Web App: Regular web applications (most common)             │
│ ├── Web App for Containers: Run Docker containers               │
│ ├── API App: REST APIs (same as Web App, just a label)         │
│ ├── Mobile App: Backend for mobile apps (deprecated)           │
│ └── Static Web App: Static sites (separate service)            │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Feature         │ App Service   │ VMs           │ Functions    │  │
│ ├────────────────┼──────────────┼──────────────┼──────────────┤  │
│ │ Management      │ Fully managed│ You manage all│ Fully managed│  │
│ │ Scaling         │ Auto-scale ✅│ Manual/VMSS  │ Auto (0-N)  │  │
│ │ OS access       │ No ❌        │ Full root ✅ │ No ❌        │  │
│ │ Custom software │ Limited      │ Anything ✅  │ Very limited │  │
│ │ Deployment      │ Git push ✅  │ SSH + scripts│ Git push ✅  │  │
│ │ Cost model      │ Per plan     │ Per VM       │ Per execution│  │
│ │ Cold start      │ No ✅        │ No           │ Yes (Consump)│  │
│ │ Best for        │ Web apps/APIs│ Full control │ Event-driven │  │
│ └────────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                       │
│ ⚡ AWS equivalent: Elastic Beanstalk / App Runner                   │
│ ⚡ GCP equivalent: App Engine / Cloud Run                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Web App (Full Portal Walkthrough)

```
Console → App Services → Create → Web App

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Project details ──                                               │
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-webapp-prod ▼]  [Create new]                  │
│                                                                       │
│ ── Instance details ──                                              │
│                                                                       │
│ Name: [app-mywebsite-prod]                                         │
│ ⚡ Globally unique! Creates URL: app-mywebsite-prod.azurewebsites.net│
│   Naming: app-{purpose}-{env}                                     │
│                                                                       │
│ Publish:                                                             │
│ ● Code (deploy your source code directly)                         │
│ ○ Docker Container (deploy a container image)                     │
│ ○ Static Web App (for static sites — React, Vue, Angular)       │
│                                                                       │
│ Runtime stack: [Node 20 LTS ▼]                                    │
│   Options:                                                          │
│   ├── .NET 8 (LTS) / .NET 9                                      │
│   ├── Node 18 LTS / Node 20 LTS                                  │
│   ├── Python 3.11 / Python 3.12                                   │
│   ├── Java 17 / Java 21                                            │
│   ├── PHP 8.2 / PHP 8.3                                           │
│   └── Ruby 3.2                                                     │
│                                                                       │
│ Operating System:                                                    │
│ ● Linux (recommended — cheaper, faster, more runtimes)           │
│ ○ Windows (needed for .NET Framework, classic ASP.NET)           │
│                                                                       │
│ Region: [Central India ▼]                                          │
│ ⚡ Choose region closest to your users.                             │
│   Not all plans/features available in all regions.                │
│                                                                       │
│ ── Pricing plans ──                                                 │
│                                                                       │
│ Linux Plan: [asp-mywebsite-prod ▼]  [Create new]                 │
│                                                                       │
│ Pricing plan: [Premium v3 P1v3 ▼]                                 │
│ ┌────────────────────────────────────────────────────────────┐    │
│ │ Free F1:      60 min/day compute, no custom domain, no SSL │    │
│ │ Basic B1:     $13/mo, 1 core, 1.75 GB, manual scale only │    │
│ │ Standard S1:  $70/mo, 1 core, 1.75 GB, auto-scale, slots │    │
│ │ Premium P1v3: $110/mo, 2 cores, 8 GB, faster, VNet, slots│    │
│ │ Premium P2v3: $220/mo, 4 cores, 16 GB                     │    │
│ │ Premium P3v3: $440/mo, 8 cores, 32 GB                     │    │
│ │ Isolated I1v2:$280/mo, dedicated env (ASE), compliance   │    │
│ └────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ ⚡ Which plan to pick:                                               │
│   Dev/test → Free F1 or Basic B1                                 │
│   Small production → Standard S1 (auto-scale + slots)           │
│   Production → Premium P1v3 (best value, VNet, faster)         │
│   Enterprise/compliance → Isolated v2 (dedicated environment) │
│                                                                       │
│ [Next: Database >]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: DATABASE (optional)                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ☐ Create a new database for your web app                           │
│                                                                       │
│ (If checked:)                                                       │
│ Database engine:                                                     │
│   ○ Azure SQL Database                                             │
│   ○ Azure Database for PostgreSQL - Flexible Server              │
│   ○ Azure Database for MySQL - Flexible Server                   │
│   ○ Azure Cosmos DB (for MongoDB)                                 │
│                                                                       │
│ ⚡ Convenient for quick setup. Creates database + connection string│
│   automatically in App Settings. Same as creating DB separately. │
│                                                                       │
│ [Next: Deployment >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: DEPLOYMENT                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── GitHub Actions settings ──                                       │
│                                                                       │
│ Continuous deployment: ● Disable  ○ Enable                        │
│                                                                       │
│ (If Enable:)                                                        │
│ GitHub account: [Connect to GitHub]                                │
│ Organization: [your-org ▼]                                        │
│ Repository: [my-webapp ▼]                                         │
│ Branch: [main ▼]                                                  │
│                                                                       │
│ ⚡ This creates a GitHub Actions workflow file in your repo.       │
│   Every push to main → automatically deploys to App Service.    │
│   File: .github/workflows/azure-deploy.yml                       │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 4: NETWORKING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Enable public access: ● On  ○ Off                                 │
│ ⚡ Off = only accessible via private endpoints.                    │
│                                                                       │
│ Enable network injection:                                           │
│ ☐ Enable VNet integration                                         │
│                                                                       │
│ (If checked:)                                                       │
│ Virtual Network: [vnet-prod ▼]                                    │
│ Subnet: [subnet-webapp (10.0.3.0/24) ▼]                         │
│                                                                       │
│ ⚡ VNet integration allows your app to access PRIVATE resources:  │
│   ├── Private database endpoints                                 │
│   ├── Internal APIs in the VNet                                  │
│   ├── On-premises resources via VPN/ExpressRoute                │
│   └── Requires Standard or Premium plan                         │
│                                                                       │
│ [Next: Monitoring >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 5: MONITORING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Enable Application Insights: ☑ Yes                                 │
│ Application Insights: [appi-mywebsite-prod ▼]  [Create new]     │
│                                                                       │
│ ⚡ Application Insights provides:                                   │
│   ├── Request tracking (response times, failures)               │
│   ├── Dependency tracking (DB calls, HTTP calls)                │
│   ├── Exception logging with stack traces                       │
│   ├── Performance counters (CPU, memory)                        │
│   ├── Custom metrics and events                                 │
│   └── Live metrics stream                                       │
│                                                                       │
│ [Next: Tags >]                                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 6: TAGS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name            │ Value          │ Resource                         │
│ environment     │ production     │ All                              │
│ team            │ backend        │ All                              │
│ project         │ mywebsite      │ All                              │
│ cost-center     │ CC-1234        │ All                              │
│                                                                       │
│ [Review + create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: App Service Plans (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP SERVICE PLAN TIERS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┬───────┬────────┬────────┬──────────┬─────────────┐  │
│ │ Tier     │ SKU   │ Cores  │ RAM    │ Storage  │ Monthly Cost │  │
│ ├──────────┼───────┼────────┼────────┼──────────┼─────────────┤  │
│ │ Free     │ F1    │ Shared │ 1 GB   │ 1 GB     │ $0           │  │
│ │ Shared   │ D1    │ Shared │ 1 GB   │ 1 GB     │ $10          │  │
│ │ Basic    │ B1    │ 1      │ 1.75 GB│ 10 GB    │ $13          │  │
│ │ Basic    │ B2    │ 2      │ 3.5 GB │ 10 GB    │ $26          │  │
│ │ Basic    │ B3    │ 4      │ 7 GB   │ 10 GB    │ $52          │  │
│ │ Standard │ S1    │ 1      │ 1.75 GB│ 50 GB    │ $70          │  │
│ │ Standard │ S2    │ 2      │ 3.5 GB │ 50 GB    │ $140         │  │
│ │ Standard │ S3    │ 4      │ 7 GB   │ 50 GB    │ $280         │  │
│ │ Premium  │ P0v3  │ 1      │ 4 GB   │ 250 GB   │ $60          │  │
│ │ Premium  │ P1v3  │ 2      │ 8 GB   │ 250 GB   │ $110         │  │
│ │ Premium  │ P2v3  │ 4      │ 16 GB  │ 250 GB   │ $220         │  │
│ │ Premium  │ P3v3  │ 8      │ 32 GB  │ 250 GB   │ $440         │  │
│ │ Isolated │ I1v2  │ 2      │ 8 GB   │ 1 TB     │ $280         │  │
│ │ Isolated │ I2v2  │ 4      │ 16 GB  │ 1 TB     │ $560         │  │
│ │ Isolated │ I3v2  │ 8      │ 32 GB  │ 1 TB     │ $1120        │  │
│ └──────────┴───────┴────────┴────────┴──────────┴─────────────┘  │
│                                                                       │
│ Feature availability by tier:                                        │
│ ┌──────────────────┬──────┬───────┬─────┬──────┬────────┬──────┐ │
│ │ Feature          │ Free │ Shared│Basic│ Std  │Premium │ Iso  │ │
│ ├──────────────────┼──────┼───────┼─────┼──────┼────────┼──────┤ │
│ │ Custom domain    │ ❌   │ ✅   │ ✅  │ ✅  │ ✅     │ ✅  │ │
│ │ SSL binding      │ ❌   │ ❌   │ ✅  │ ✅  │ ✅     │ ✅  │ │
│ │ Auto-scale       │ ❌   │ ❌   │ ❌  │ ✅  │ ✅     │ ✅  │ │
│ │ Deployment slots │ ❌   │ ❌   │ ❌  │ 5    │ 20     │ 20  │ │
│ │ VNet integration │ ❌   │ ❌   │ ❌  │ ❌  │ ✅     │ ✅  │ │
│ │ Private endpoint │ ❌   │ ❌   │ ❌  │ ❌  │ ✅     │ ✅  │ │
│ │ Daily backups    │ ❌   │ ❌   │ ❌  │ 10/d │ 50/d   │ 50/d│ │
│ │ Always On        │ ❌   │ ❌   │ ✅  │ ✅  │ ✅     │ ✅  │ │
│ │ Max instances    │ -    │ -    │ 3   │ 10   │ 30     │ 100 │ │
│ │ SLA              │ N/A  │ N/A  │ N/A │99.95%│ 99.95% │99.95%│ │
│ └──────────────────┴──────┴───────┴─────┴──────┴────────┴──────┘ │
│                                                                       │
│ ⚡ Key difference Premium vs Standard:                               │
│   Premium has VNet integration, private endpoints, more instances,│
│   bigger storage, zone redundancy, and faster hardware.           │
│                                                                       │
│ Multiple apps on one plan:                                           │
│ ├── You CAN run multiple Web Apps on the same plan               │
│ ├── They share CPU and memory of the plan                        │
│ ├── Saves money if apps are small                                │
│ ├── Risky if one app consumes all resources                      │
│ └── Best practice: Separate plans for production apps            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Deployment Slots

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT SLOTS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What are deployment slots?                                           │
│ ├── Live instances of your app with different URLs               │
│ ├── Each slot has its own configuration (app settings, etc.)    │
│ ├── You can SWAP slots = zero-downtime deployment                │
│ └── Available on Standard tier and above                         │
│                                                                       │
│ How it works (swap for zero-downtime deploy):                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ BEFORE SWAP:                                                 │  │
│ │ ┌─────────────────┐     ┌──────────────────────┐          │  │
│ │ │ Production Slot  │     │ Staging Slot          │          │  │
│ │ │ app.azurewebsites│     │ app-staging.azureweb  │          │  │
│ │ │ Code: v1.0       │     │ Code: v2.0 (new!)     │          │  │
│ │ │ ← Users here    │     │ ← Testers here       │          │  │
│ │ └─────────────────┘     └──────────────────────┘          │  │
│ │                                                              │  │
│ │ [SWAP] ← Click swap button                                 │  │
│ │                                                              │  │
│ │ AFTER SWAP:                                                  │  │
│ │ ┌─────────────────┐     ┌──────────────────────┐          │  │
│ │ │ Production Slot  │     │ Staging Slot          │          │  │
│ │ │ app.azurewebsites│     │ app-staging.azureweb  │          │  │
│ │ │ Code: v2.0 ✅    │     │ Code: v1.0 (backup)  │          │  │
│ │ │ ← Users see new │     │ ← Old code (rollback)│          │  │
│ │ └─────────────────┘     └──────────────────────┘          │  │
│ │                                                              │  │
│ │ ⚡ If v2.0 has issues → swap again to rollback instantly!  │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Portal: App Service → Deployment slots → Add Slot                │
│                                                                       │
│ Slot name: [staging]                                                │
│ Clone settings from: [production ▼] or [Do not clone]            │
│                                                                       │
│ ⚡ Slot-specific settings:                                          │
│   Some settings should STAY with the slot (not swap):            │
│   ├── Database connection strings (staging DB ≠ prod DB)       │
│   ├── Feature flags (staging might have experimental features) │
│   ├── API keys for different environments                      │
│   └── Mark as "Deployment slot setting" ☑ to stick to slot    │
│                                                                       │
│ Common slot patterns:                                                │
│ ├── prod + staging (most common — test, then swap)              │
│ ├── prod + staging + dev (three environments)                   │
│ └── prod + canary (gradual traffic shift)                       │
│                                                                       │
│ Auto-swap (optional):                                                │
│   When code is deployed to staging → automatically swap to prod │
│   Useful for CI/CD where staging is just a deployment target.   │
│                                                                       │
│ Traffic routing (preview/canary):                                    │
│   Route a % of production traffic to staging:                    │
│   Production: 90%  →  Staging: 10%                               │
│   Gradually increase staging % if everything is healthy.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Console: Managing Deployment Slots

```
Portal → App Service → Deployment slots

┌─────────────────────────────────────────────────────────────────────┐
│ Deployment Slots                                           [+ Add] │
├────────────────────────┬──────────┬───────────┬───────────────────┤
│ Name                   │ Status   │ Traffic % │ URL               │
├────────────────────────┼──────────┼───────────┼───────────────────┤
│ production (default)   │ Running  │ 90%       │ app.azurewebsites │
│ staging                │ Running  │ 10%       │ app-staging.azure │
└────────────────────────┴──────────┴───────────┴───────────────────┘

Actions:
├── [Swap] → Swap production ↔ staging
├── [Start/Stop] → Start or stop a slot independently
├── [Delete] → Remove a slot (cannot delete production)
└── [Clone] → Copy settings from another slot
```

---

## Part 5: Custom Domains & SSL

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM DOMAINS & SSL                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal → App Service → Custom domains → [+ Add custom domain]    │
│                                                                       │
│ Step 1: Add your domain                                             │
│ Domain: [www.mywebsite.com]                                        │
│                                                                       │
│ Step 2: Validate ownership (DNS records)                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ For www.mywebsite.com:                                       │  │
│ │ Type: CNAME                                                  │  │
│ │ Name: www                                                    │  │
│ │ Value: app-mywebsite-prod.azurewebsites.net                 │  │
│ │                                                              │  │
│ │ For mywebsite.com (root/apex domain):                       │  │
│ │ Type: A record                                               │  │
│ │ Name: @                                                      │  │
│ │ Value: <your app's IP address>                              │  │
│ │ + TXT record for verification                               │  │
│ │ Name: asuid                                                  │  │
│ │ Value: <verification ID from Azure>                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Step 3: SSL Certificate                                             │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Option 1: Free Managed Certificate (recommended!)            │  │
│ │   ├── Azure creates and renews SSL cert automatically       │  │
│ │   ├── No cost, no expiry worries                            │  │
│ │   ├── Only for custom domains (not apex with A record)     │  │
│ │   └── Limited to single domain (no wildcard)               │  │
│ │                                                              │  │
│ │ Option 2: App Service Certificate (paid)                     │  │
│ │   ├── $70/year for standard, $300/year for wildcard        │  │
│ │   ├── Supports apex domain and wildcard (*.mysite.com)     │  │
│ │   ├── Auto-renewal                                          │  │
│ │   └── Stored in Key Vault                                   │  │
│ │                                                              │  │
│ │ Option 3: Upload your own certificate (.pfx file)           │  │
│ │   ├── Bring certs from Let's Encrypt, DigiCert, etc.      │  │
│ │   ├── You manage renewal                                    │  │
│ │   └── Upload via Portal or Key Vault reference             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ HTTPS enforcement:                                                   │
│ Portal → App Service → TLS/SSL settings                          │
│ HTTPS Only: ● On (redirects HTTP to HTTPS — ALWAYS enable!)     │
│ Minimum TLS Version: [1.2 ▼] (recommended minimum)              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Scaling (Up & Out)

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCALING                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two types of scaling:                                                │
│                                                                       │
│ Scale UP (vertical) = Bigger machine                                │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ B1 (1 core, 1.75 GB) → P1v3 (2 cores, 8 GB)              │  │
│ │ ⚡ Change the App Service Plan SKU                           │  │
│ │   Portal → App Service Plan → Scale up                     │  │
│ │   No downtime! Azure migrates your app to bigger hardware. │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Scale OUT (horizontal) = More instances                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ 1 instance → 3 instances (each handles part of traffic)    │  │
│ │                                                              │  │
│ │ Manual scale:                                                │  │
│ │   Portal → App Service Plan → Scale out                    │  │
│ │   Instance count: [3]  (slider or input)                   │  │
│ │                                                              │  │
│ │ Auto-scale (Standard and above):                             │  │
│ │   Portal → App Service Plan → Scale out → Custom autoscale│  │
│ │                                                              │  │
│ │   Rule example:                                              │  │
│ │   ┌────────────────────────────────────────────────────┐   │  │
│ │   │ When: CPU% > 70% for 10 minutes                    │   │  │
│ │   │ Action: Increase instance count by 1               │   │  │
│ │   │ Cooldown: 5 minutes                                 │   │  │
│ │   │                                                      │   │  │
│ │   │ When: CPU% < 30% for 10 minutes                    │   │  │
│ │   │ Action: Decrease instance count by 1               │   │  │
│ │   │ Cooldown: 5 minutes                                 │   │  │
│ │   │                                                      │   │  │
│ │   │ Limits: Min 1, Max 10, Default 2                   │   │  │
│ │   └────────────────────────────────────────────────────┘   │  │
│ │                                                              │  │
│ │   Available metrics for auto-scale:                          │  │
│ │   ├── CPU Percentage                                        │  │
│ │   ├── Memory Percentage                                     │  │
│ │   ├── HTTP Queue Length                                     │  │
│ │   ├── Data In / Data Out                                   │  │
│ │   ├── Disk Queue Length                                     │  │
│ │   └── Custom metrics (from Application Insights)           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Azure auto-scale distributes traffic across instances using    │  │
│   the built-in load balancer. You don't configure LB separately.│  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Configuration & App Settings

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP SETTINGS & CONFIGURATION                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal → App Service → Configuration                              │
│                                                                       │
│ ── Application settings ──                                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ These are environment variables available to your app.       │  │
│ │                                                              │  │
│ │ Name                    │ Value            │ Slot setting   │  │
│ │ DATABASE_URL            │ ●●●●●●           │ ☑             │  │
│ │ REDIS_URL               │ ●●●●●●           │ ☑             │  │
│ │ API_KEY                 │ ●●●●●●           │ ☑             │  │
│ │ NODE_ENV                │ production       │ ☑             │  │
│ │ FEATURE_FLAG_NEW_UI     │ true             │ ☐             │  │
│ │                                                              │  │
│ │ ⚡ Values are hidden (shown as ●●●●●●) for security.       │  │
│ │   Click "Show values" to reveal.                           │  │
│ │                                                              │  │
│ │ ⚡ "Slot setting" checkbox:                                  │  │
│ │   ☑ = Value stays with this slot (doesn't swap)           │  │
│ │   ☐ = Value swaps with the code                           │  │
│ │                                                              │  │
│ │ ⚡ Better approach: Use Key Vault references!               │  │
│ │   Value: @Microsoft.KeyVault(SecretUri=https://...)        │  │
│ │   App reads secret from Key Vault at runtime.              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Connection strings ──                                            │
│ (Legacy — use App Settings instead for new apps)                   │
│                                                                       │
│ ── General settings ──                                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Stack settings:                                              │  │
│ │   Major version: [Node 20 LTS]                              │  │
│ │   Minor version: [20.x]                                     │  │
│ │   Startup command: [npm start] (optional override)         │  │
│ │                                                              │  │
│ │ Platform settings:                                           │  │
│ │   Platform: [64 bit ▼]                                     │  │
│ │   FTP state: [FTPS Only ▼]  (or Disabled — recommended)  │  │
│ │   HTTP version: [2.0 ▼]                                    │  │
│ │   Web sockets: ○ On  ● Off                                 │  │
│ │   Always On: ● On  ○ Off                                   │  │
│ │   ⚡ Always On: Keeps app loaded even without traffic.     │  │
│ │     If Off, app unloads after 20 min idle (slow restart).  │  │
│ │     Always enable for production!                          │  │
│ │   ARR affinity: ○ On  ● Off                                │  │
│ │   ⚡ ARR = Sticky sessions. Off recommended (stateless).  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Path mappings ──                                                 │
│ Configure mount points for Azure Storage (files/blobs).           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Deployment Methods

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT METHODS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. GitHub Actions (recommended for teams)                           │
│    ├── Azure creates workflow file in your repo                   │
│    ├── Push to branch → auto-deploy                              │
│    └── Portal → Deployment Center → Source: GitHub               │
│                                                                       │
│ 2. Azure DevOps Pipelines                                           │
│    ├── YAML pipeline with AzureWebApp@1 task                     │
│    └── Portal → Deployment Center → Source: Azure Repos          │
│                                                                       │
│ 3. Local Git                                                         │
│    ├── Push directly to Azure Git remote                          │
│    ├── git remote add azure https://<app>.scm.azurewebsites.net │
│    └── git push azure main                                       │
│                                                                       │
│ 4. ZIP Deploy (CLI)                                                  │
│    ├── az webapp deploy --resource-group rg --name app --src app.zip│
│    └── Good for CI/CD pipelines                                  │
│                                                                       │
│ 5. FTP/FTPS (not recommended)                                       │
│    └── Legacy. Use other methods instead.                         │
│                                                                       │
│ 6. Docker Container                                                  │
│    ├── Publish: Docker Container → point to ACR/Docker Hub      │
│    ├── Portal → Deployment Center → Source: Container Registry  │
│    └── Supports continuous deployment (webhook on image push)   │
│                                                                       │
│ Portal → App Service → Deployment Center:                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Source: [GitHub ▼]                                           │  │
│ │ Organization: [my-org]                                       │  │
│ │ Repository: [my-app]                                         │  │
│ │ Branch: [main]                                               │  │
│ │ Build provider: [GitHub Actions ▼]                          │  │
│ │                                                              │  │
│ │ [Save] → Creates .github/workflows/deploy.yml in your repo │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Networking & Access Restrictions

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal → App Service → Networking                                 │
│                                                                       │
│ ── Inbound traffic ──                                               │
│                                                                       │
│ Access Restrictions:                                                 │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Restrict who can access your app:                            │  │
│ │                                                              │  │
│ │ Rule 1: Allow office IP                                      │  │
│ │   Priority: 100                                              │  │
│ │   Action: Allow                                              │  │
│ │   Source: IP Address: 203.0.113.0/24                        │  │
│ │                                                              │  │
│ │ Rule 2: Allow VNet subnet                                    │  │
│ │   Priority: 200                                              │  │
│ │   Action: Allow                                              │  │
│ │   Source: Virtual Network: vnet-prod/subnet-frontend        │  │
│ │                                                              │  │
│ │ Rule 3: Allow Front Door                                     │  │
│ │   Priority: 300                                              │  │
│ │   Action: Allow                                              │  │
│ │   Source: Service Tag: AzureFrontDoor.Backend               │  │
│ │   X-Azure-FDID header: <your-front-door-id>               │  │
│ │                                                              │  │
│ │ Default: Deny all (unmatched traffic is blocked)            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Private Endpoint (Premium plan):                                    │
│ ├── Makes app accessible ONLY from your VNet                    │
│ ├── No public URL exposure                                      │
│ ├── Access via private IP (10.0.x.x)                           │
│ └── Requires Private DNS Zone for name resolution               │
│                                                                       │
│ ── Outbound traffic ──                                              │
│                                                                       │
│ VNet Integration (Premium plan):                                    │
│ ├── App can access private resources in VNet                    │
│ ├── Private database endpoints, internal APIs                   │
│ ├── On-premises via VPN Gateway or ExpressRoute                │
│ └── Subnet must be delegated to Microsoft.Web/serverFarms      │
│                                                                       │
│ Hybrid Connections:                                                  │
│ ├── Access on-premises resources without VPN                    │
│ ├── Uses relay agent installed on-premises                      │
│ └── TCP tunnel through Azure Relay                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Terraform & Bicep

### Terraform

```hcl
# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "asp-mywebsite-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = "P1v3"
}

# Web App
resource "azurerm_linux_web_app" "main" {
  name                = "app-mywebsite-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    always_on = true

    application_stack {
      node_version = "20-lts"
    }
  }

  app_settings = {
    "NODE_ENV"     = "production"
    "DATABASE_URL" = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.db_url.id})"
  }

  identity {
    type = "SystemAssigned"
  }
}

# Staging slot
resource "azurerm_linux_web_app_slot" "staging" {
  name           = "staging"
  app_service_id = azurerm_linux_web_app.main.id

  site_config {
    always_on = true
    application_stack {
      node_version = "20-lts"
    }
  }
}
```

### Bicep

```bicep
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-mywebsite-prod'
  location: location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
  }
  kind: 'linux'
  properties: {
    reserved: true  // Required for Linux
  }
}

resource webApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'app-mywebsite-prod'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      alwaysOn: true
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

---

## Part 11: az CLI Reference

```bash
# Create App Service Plan
az appservice plan create \
  --name asp-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --location centralindia \
  --sku P1v3 \
  --is-linux

# Create Web App
az webapp create \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --plan asp-mywebsite-prod \
  --runtime "NODE:20-lts"

# Configure app settings
az webapp config appsettings set \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --settings NODE_ENV=production API_KEY=@Microsoft.KeyVault(SecretUri=https://...)

# Create deployment slot
az webapp deployment slot create \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --slot staging

# Deploy code (ZIP)
az webapp deploy \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --src-path ./app.zip

# Swap slots
az webapp deployment slot swap \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --slot staging \
  --target-slot production

# Add custom domain
az webapp config hostname add \
  --webapp-name app-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --hostname www.mywebsite.com

# Enable managed certificate (free SSL)
az webapp config ssl create \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --hostname www.mywebsite.com

# Scale out (manual)
az appservice plan update \
  --name asp-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --number-of-workers 3

# View logs (live tail)
az webapp log tail \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod

# Restart app
az webapp restart \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod

# Delete web app
az webapp delete \
  --name app-mywebsite-prod \
  --resource-group rg-webapp-prod

# Delete App Service Plan
az appservice plan delete \
  --name asp-mywebsite-prod \
  --resource-group rg-webapp-prod \
  --yes
```

---

## Part 12: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: Production Web App with CI/CD                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GitHub repo (main branch)                                           │
│       ↓ (push triggers GitHub Actions)                             │
│ GitHub Actions: build → test → deploy to staging slot            │
│       ↓ (approval gate)                                            │
│ Swap staging → production (zero downtime)                         │
│                                                                       │
│ App Service (P1v3 Linux):                                           │
│ ├── Slot: production (custom domain + managed SSL)               │
│ ├── Slot: staging (auto-deploy from GitHub)                      │
│ ├── VNet integration → private DB endpoint                      │
│ ├── Application Insights → monitoring & alerts                  │
│ ├── Auto-scale: 2-10 instances based on CPU                     │
│ └── Key Vault references for secrets                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: Multi-Region with Front Door                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure Front Door (global load balancer + CDN)                      │
│ ├── Origin 1: app-prod-eastus (East US)                          │
│ ├── Origin 2: app-prod-westeu (West Europe)                     │
│ └── WAF policy (OWASP rules)                                     │
│                                                                       │
│ Each App Service:                                                    │
│ ├── Access restricted to Front Door only (service tag)           │
│ ├── Connected to regional database (read replica)                │
│ └── Auto-scale independently per region                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
App Service = PaaS for web apps (no server management)
App Service Plan = The compute (CPU/RAM) your apps run on
Deployment Slot = Staging environment for zero-downtime deploys
Always On = Keep app loaded (enable for production)
VNet Integration = Access private resources (Premium plan)
Access Restrictions = IP/VNet-based firewall for inbound traffic
Free Managed Cert = Free SSL certificate (auto-renewed)
ARR Affinity = Sticky sessions (disable for stateless apps)

Plan selection:
  Dev/test → Free F1 or Basic B1
  Small prod → Standard S1 (auto-scale + 5 slots)
  Production → Premium P1v3 (VNet, 20 slots, zone redundancy)
  Enterprise → Isolated I1v2 (dedicated, compliance)
```

---

## What's Next?

Next chapter: [Chapter 21: Azure Container Apps](21-container-apps.md) — Deploy containers that auto-scale (including to zero) without managing Kubernetes infrastructure.
